# サーバー構築

## 前提条件

- **Node.js 24 必須**（v0.5.0〜）
- Docker / Docker Compose
- Git

!!! warning "Node 24 未満では `npm install` が hard fail します"
    v0.5.0（[#330](https://github.com/gamasenninn/tealus/issues/330)）でコードベースが全レイヤー TypeScript 化され、サーバー系（server / agent-server / rtc-server）は **`.mts` をビルドせず直接実行**します（`node src/app.mts`、トランスパイル工程なし）。これは Node の native type stripping に依存する構成で、**型注釈をそのまま解釈できない版では起動しません**。

    **検証されているのは Node 24 だけです。** Dockerfile はすべて `node:24` ベース、CI も全ジョブが Node 24 で走っています。

    全 package.json の `engines.node` が `>=24.0.0`、ルート `.npmrc` に `engine-strict=true` が設定されているため、**24 未満では `npm install` 自体が `EBADENGINE` で hard fail** します。これは意図的な設計です — **install が通って起動時に落ちる（＝「入ったのに動かない」で原因に辿り着けない）よりも、入口で止める**方針です（[#210](https://github.com/gamasenninn/tealus/issues/210) 以来の踏襲）。

    Node 24 のインストールには nvm（推奨）または NodeSource パッケージを利用してください。

## 1. リポジトリのクローン

```bash
git clone https://github.com/gamasenninn/tealus.git
cd tealus
```

## 2. Docker 起動（開発用: DB のみ）

```bash
docker-compose up -d
```

以下のサービスが起動します。アプリケーション本体（server / agent-server）はホスト側で `npm run dev` する開発フロー想定です。

| サービス | ポート | 用途 |
|---|---|---|
| PostgreSQL | 5432 | 開発用DB（★ `127.0.0.1` でのみ待ち受け） |
| PostgreSQL | 5433 | テスト用DB（★ `127.0.0.1` でのみ待ち受け） |

## 2.5 Docker フルデプロイ（本番想定）

v0.1.x（[#188](https://github.com/gamasenninn/tealus/issues/188)）から、server / agent-server / rtc-server を含む全サービスを Docker で一括起動できるようになりました。3 構成から運用形態に合わせて選択します。

### 3 つの構成

| 構成 | 起動コマンド | 含まれるサービス | 対応OS |
|---|---|---|---|
| 開発（DB のみ） | `docker-compose up -d` | postgres / postgres-test | Mac / Windows / Linux |
| **フル**（server + agent 含む） | `docker-compose -f docker-compose.full.yml up -d` | 上記 + server:3000 + agent-server:4000 | Mac / Windows / Linux |
| + RTC（通話・legacy TTS） | `docker-compose -f docker-compose.full.yml -f docker-compose.rtc.yml up -d` | 上記 + rtc-server:3100 | **Linux のみ** |

!!! warning "★ コードを更新したら `--build` が要ります"
    `docker-compose ... up -d` だけでは**イメージが作り直されず、古いコードのまま起動します**（[#558](https://github.com/gamasenninn/tealus/issues/558)）。更新時は `up -d --build` を使ってください。

!!! info "初回起動時の migration"
    PostgreSQL の initdb は migration を流しません。**server の起動時に実行されます**（[#557](https://github.com/gamasenninn/tealus/issues/557)）ので、初回は server のログで `[migration-check]` を確認してください。

!!! note "Redis は使いません"
    以前このページには Redis が構成に含まれると書いていましたが、**誤りでした**。Redis は初期設計の名残で**一度もコードから使われておらず**、[#466](https://github.com/gamasenninn/tealus/issues/466) で依存ごと削除されています。在席状態は Socket.IO が内部で持ち、LINE の画像の束ね・ログイン試行の制限はいずれもプロセス内のメモリで処理しています（「1 台・1 プロセス」構成が前提）。

!!! info "TTS のみなら rtc-server は不要"
    v0.1.x の default TTS 配信は Socket.IO blob 経由のため、TTS だけ使うなら **+RTC 構成は不要**です。WebRTC 通話機能を使う場合、または `TTS_BROADCAST_MEDIASOUP=true` で legacy mediasoup TTS を使う場合のみ rtc-server が必要になります。

### 各 Dockerfile の概要

| サービス | ベースイメージ | 構成 | 主な特徴 |
|---|---|---|---|
| server | `node:24-alpine` | 3 段階 multi-stage（client build → dashboard build → runtime） | client/dist と dashboard/dist を同梱 |
| agent-server | `node:24-alpine` | 単段ビルド | webhook + AI agent のみで軽量 |
| rtc-server | `node:24-bookworm-slim` | 2 段階 multi-stage（builder bookworm-slim → runtime bookworm-slim） | mediasoup の native worker が glibc 必須のため bookworm（Debian）ベース |

!!! note "イメージサイズの記載について"
    以前ここに記載していた最終イメージサイズ（~312MB / ~261MB / ~307MB）は **v0.1.x 当時・Node 20 ベースでの計測値**です。v0.5.0 のベースイメージ更新後は再計測していないため、記載を取り下げています。

### MEDIA_ROOT の bind mount

アップロードファイル（画像・動画・音声）の保存先は環境変数 `MEDIA_ROOT` で指定します。Docker 運用時は **ホスト側の永続パスを bind mount** することで、コンテナ再作成後もファイルが保持されます。

```yaml
# docker-compose.full.yml の例
services:
  server:
    volumes:
      - ./media:/app/media   # ホストの ./media をコンテナ /app/media に bind
    environment:
      MEDIA_ROOT: /app/media
```

### rtc-server が Linux 限定の理由

rtc-server は以下 2 つの理由で Linux ホストでのみ動作します。

1. **mediasoup の native worker が glibc 必須**: musl ベースの Alpine では動作不可のため、Dockerfile も bookworm-slim を使用
2. **`network_mode: host` 必須**: WebRTC の UDP ポート 10000-10100 を NAT 越えなしに公開する必要があり、Docker の host network mode が利用できる Linux 限定

Mac / Windows ホストで通話機能を使いたい場合は、別途 Linux ホストに rtc-server を立てる構成（または開発時は `npm run dev` でホストに直接起動）を検討してください。

## 3. サーバーセットアップ

```bash
cd server
npm install
```

### 環境変数の設定

`server/.env` にはデフォルト値が設定されていますが、本番環境では以下を必ず変更してください。

| 変数 | 説明 | 注意事項 |
|---|---|---|
| `JWT_SECRET` | JWT署名キー | **必ず変更**。推測困難な文字列を設定 |
| `VAPID_PUBLIC_KEY` | Web Push公開鍵 | 下記コマンドで生成 |
| `VAPID_PRIVATE_KEY` | Web Push秘密鍵 | 下記コマンドで生成 |
| `OPENAI_API_KEY` | OpenAI APIキー | 音声文字起こし・スタンプ生成に必要 |
| `DB_PASSWORD` | データベースパスワード | 本番では強力なパスワードに変更 |

#### VAPID鍵の生成

```bash
npx web-push generate-vapid-keys
```

### 全環境変数一覧 {#all-env}

!!! danger "本番運用前の必須変更"
    下表の **「開発用デフォルト」列は localhost 検証用の値** です。**本番ではすべて固有の値に置換してください**。特に `JWT_SECRET` / `DB_PASSWORD` を未変更のまま外部公開した場合、トークン偽造や DB 侵入の対象になります。

| 変数名 | 開発用デフォルト | 説明 |
|---|---|---|
| `PORT` | 3000 | サーバーポート |
| `DB_HOST` | localhost | PostgreSQLホスト |
| `DB_PORT` | 5432 | PostgreSQLポート |
| `DB_NAME` | tealus | データベース名 |
| `DB_USER` | tealus | データベースユーザー |
| `DB_PASSWORD` | **本番では必ず固有値に変更** | データベースパスワード |
| `JWT_SECRET` | **本番では必ず固有値を生成** （`openssl rand -hex 32` 等）| JWT署名シークレット |
| `VAPID_PUBLIC_KEY` | — | Web Push公開鍵（`npx web-push generate-vapid-keys` で生成） |
| `VAPID_PRIVATE_KEY` | — | Web Push秘密鍵（同上） |
| `VAPID_SUBJECT` | — | Web Push の連絡先。★ **実在するメールアドレス（`mailto:`）か `https:` の URL** にしてください。実在しないドメインだと **iPhone への通知が 403（BadJwtToken）で断られます**。未設定・形式違いは起動時に警告が出ます |
| `OPENAI_API_KEY` | — | OpenAI APIキー |
| `MEDIA_ROOT` | ../../media | アップロードファイル保存先 |
| `AGENT_PORT` | 4000 | エージェントサーバーポート |
| `RTC_PORT` | 3100 | mediasoupシグナリングポート（rtc-server を使う場合） |
| `TTS_BROADCAST_MEDIASOUP` | `false` | TTS を mediasoup 経由で配信（legacy）。`true` で v0.1.0 以前の rtc-server 経由配信に切替（[#189](https://github.com/gamasenninn/tealus/issues/189)） |
| `LOG_LEVEL` | debug（開発） / info（本番） | ログレベル |
| `NODE_ENV` | — | ★ **本番は `production` を推奨**（[#603](https://github.com/gamasenninn/tealus/issues/603)）。`JWT_SECRET` 未設定なら**起動を止める**、Socket.IO の CORS を閉じる、記録の既定を `info` にする。今までどおり `[debug]` 行を残すなら `LOG_LEVEL=debug` も |
| `TRUST_PROXY` | — | ★ **リバースプロキシ（Nginx / Cloudflare 等）の内側で動かすなら設定**（[#362](https://github.com/gamasenninn/tealus/issues/362)）。信頼する前段の IP（例 `loopback`）を入れると、そこから来たときだけ `X-Forwarded-For` を見る。未設定だとログインの回数制限が**全員を前段 1 つの IP として数える**（起動時に警告が出ます） |
| `STT_BACKEND` | `openai` | 文字起こしの経路（[#424](https://github.com/gamasenninn/tealus/issues/424)）。`openai`（`WHISPER_MODEL`）／`local`（自ホスト Qwen3-ASR、`QWEN_WORKER_URL`）／`gemini`（`GEMINI_API_KEY` 必須、辞書の語を渡す）。local / gemini が失敗したときは openai に切り替わる |
| `AGENT_URL` / `RTC_URL` / `MCP_HTTP_URL` | `http://localhost:{AGENT_PORT / RTC_PORT / MCP_HTTP_PORT}` | server が中継する先（[#556](https://github.com/gamasenninn/tealus/issues/556)）。agent-server・rtc-server・MCP を**別のホストやコンテナ**で動かすときに指定（Docker フルデプロイでは compose 側で設定済み） |
| `VITE_ALLOWED_HOSTS` | — | 外部ホスト経由で vite dev server に access する場合に許可ホストを指定（v0.2.0〜の注意点） |
| `TRANSCRIPTION_MODE` | — | `organon` を指定すると、自ホスト STT（Qwen3-ASR）の生出力を organon 知識で文脈補正する 2 段構成に切替（v0.5.0、[#326](https://github.com/gamasenninn/tealus/issues/326)） |
| `LINE_IMAGESET_FLUSH_MS` | **60000** | LINE の複数画像同時送信（imageSet）を 1 メッセージに束ねる待ち時間。全枚が揃えば即投稿、揃わなければ**最後の 1 枚から**この時間で届いた分だけ投稿（v0.7.0 [#353](https://github.com/gamasenninn/tealus/issues/353)、既定 15 秒 → 60 秒 [#552](https://github.com/gamasenninn/tealus/issues/552)） |
| `LINE_IMAGESET_MAX_WAIT_MS` | 300000 | 同じ束ねの**最初の 1 枚から**の上限（5 分）。画像が届き続けても、ここで区切って投稿する（[#552](https://github.com/gamasenninn/tealus/issues/552)） |

### DBマイグレーション

```bash
npm run migrate
```

マイグレーションは `server/src/db/migrations/` 内の連番SQLファイル（**001〜040**）が順に実行されます。適用済みの記録は DB の **`schema_migrations` テーブル**（台帳）に残ります（[#406](https://github.com/gamasenninn/tealus/issues/406)）。

!!! danger "★ 更新するときは、必ず migrate を**再起動の前**に"
    v0.11.0 以降、**`034` を流さずにサーバーを再起動すると、ログインそのものが失敗します**（認証の問い合わせが新しい列を読むため）。プッシュの送信も止まります。

    更新の正しい順序は「コード取得 → 依存更新 → **migrate** → client 再ビルド → 再起動」です。手順の本文は本体リポジトリの [upgrade-guide.md](https://github.com/gamasenninn/tealus/blob/main/docs/upgrade-guide.md) にあります。

!!! danger "★ 台帳より前から動かしている DB は、`--baseline-through` が要ります"
    台帳（`schema_migrations`）が導入される前から動かしている DB には適用済みの記録がないため、**通常の `npm run migrate` は止まります**。このとき案内に出る `--baseline`（上限なし）を打つと、**まだ流していない migration まで「適用済み」として記録され、静かに壊れます**（[#500](https://github.com/gamasenninn/tealus/issues/500)）。

    **どこまで適用済みかを指定して記録し、残りは通常実行で流します。** 例えば v0.9.0 相当（`026` まで適用済み）の DB では:

    ```bash
    cd server
    npm run migrate -- --baseline-through 026_message_form_type.sql   # ここまでを「適用済み」と記録（流さない）
    npm run migrate                                                   # 027 以降が実際に流れる
    ```

!!! warning "migrate 忘れは「クラッシュしないが静かに壊れる」"
    未適用の migration がある状態でも**サーバーが起動してしまう**ことがあり、機能だけが欠けます（例: 辞書テーブルが無いと辞書育成画面が 500、自己成長辞書が効かない）。起動時の確認は v0.11.1 で強化され、台帳とファイルを突き合わせて **`[migration-check] ★ 未適用 N 件: …`** とファイル名つきで出るようになりました。**起動ログの `[migration-check]` / `[build-check]`** を必ず確認してください。

!!! warning "managed PostgreSQL 環境での注意"
    migration 021 は `pg_trgm` extension を要求します。`CREATE EXTENSION` 権限がない managed PostgreSQL 環境（一部のクラウド DB）では migration が失敗するため、運用前に extension 作成権限を確認してください。

### サーバー起動

```bash
npm run dev    # 開発モード（nodemon で自動再起動）
npm start      # 本番モード
```

サーバーは `http://localhost:3000` で起動します。

## 4. クライアントセットアップ

```bash
cd client
npm install
npm run dev
```

クライアントは `http://localhost:5173` で起動します。Viteのプロキシ設定により `/api/*` と `/socket.io` は自動的にサーバーに転送されます。

### 本番ビルド

```bash
cd client
npm run build
```

`dist/` ディレクトリにビルド成果物が生成されます。

## 5. 初回ユーザー登録

ブラウザで `http://localhost:5173` を開いても、初期状態ではユーザーが存在しません。APIで管理者ユーザーを登録します。

!!! warning "パスワードについて"
    下記の `password123` は **localhost 検証用のサンプル値** です。本番運用では必ず強固なパスワード（12 文字以上、英数記号混在）に置き換えてください。

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"login_id":"admin","display_name":"管理者","password":"password123"}'
```

!!! info "最初のユーザーは自動的に admin"
    v0.2.1（[#211](https://github.com/gamasenninn/tealus/issues/211)）から、**最初の非 Bot ユーザー**（`is_bot=false` の COUNT が 0 の状態で `register` を呼んだユーザー）が自動的に `admin` role で作成されます。Mattermost / Rocket.Chat / GitLab 標準の OSS first-user-admin パターンです。

!!! danger "★ この curl が通るのは 1 回だけです"
    **自己登録は「人の利用者が 0 人のとき」しか開いていません**（[#469](https://github.com/gamasenninn/tealus/issues/469)）。1 人でも登録済みだと、同じ curl は **403** を返します。以前このページは「2 人目以降もこの方法で」と読める書き方をしていましたが、**その手順では作れません**。

    **2 人目以降は、管理者でログインして `/admin` のユーザー管理から作成**してください。Bot ユーザーも同じ画面で作れます（DB を直接編集する必要はありません）。

以降はログイン画面からユーザーIDとパスワードでログインできます。

## 6. AIエージェントサーバー（オプション）

AI機能を使用する場合は、エージェントサーバーも起動します。

```bash
cd agent-server
npm install
npm run dev
```

エージェントサーバーは `http://localhost:4000` で起動します。

### エージェント環境変数

`agent-server/.env` に以下を設定します。

| 変数名 | 説明 |
|---|---|
| `TEALUS_API_URL` | Tealus本体のURL（例: `http://localhost:3000`） |
| **`TEALUS_BOT_ID`** | Bot ユーザーのログイン ID（★ `TEALUS_USER_ID` ではありません） |
| **`TEALUS_BOT_PASS`** | Bot ユーザーのパスワード（★ `TEALUS_PASSWORD` ではありません） |
| **`JWT_SECRET`** | **本体と同じ値**。agent-server も同じトークンで認証するため必須 |
| `WEBHOOK_SECRET` | webhook の署名検証用。**本体の Webhook 登録に入れた secret と同じ値**（[#478](https://github.com/gamasenninn/tealus/issues/478)） |
| `OPENAI_API_KEY` | OpenAI APIキー |
| `AGENT_WORKSPACE_ROOT` | ワークスペースのパス |
| `AIVIS_API_KEY` | Aivis Cloud API キー（TTS 音声合成用） |
| `AIVIS_MODEL_UUID` | デフォルト音声モデル UUID（デフォルト: 凛音エル） |
| `TTS_ENABLED` | TTS 読み上げの有効/無効（`"false"` で無効化、デフォルト: 有効） |
| `TTS_MAX_LENGTH` | 読み上げ最大文字数（デフォルト: 500） |
| `WHISPER_MODEL` | 音声文字起こしモデル（default **`gpt-4o-mini-transcribe`**、[#284](https://github.com/gamasenninn/tealus/issues/284)。`gpt-4o-transcribe` は無音やノイズで業務無線らしい文を作る幻覚が出たため既定から外した） |
| `ORGANON_INJECT` | organon の `sql_mapping` を agent prompt に注入（**default OFF の opt-in**、v0.4.0 [#304](https://github.com/gamasenninn/tealus/issues/304)。旧 `INJECT_ORGANON_POLYSEME` は廃止・fallback 無し） |
| `ORGANON_REPO_PATH` | organon repo の path override |
| `VOCAB_INJECT` | 業務語彙辞書（別名→正規名）を Light / Deep の prompt にも注入し、画像 OCR・帳票読み取りの表記揺れを正規化（**default OFF の opt-in**、v0.5.0 [#315](https://github.com/gamasenninn/tealus/issues/315)） |
| `DEEP_TIMEOUT` | Deep agent のタイムアウト（**default 480000 = 8 分**、v0.5.0 [#314](https://github.com/gamasenninn/tealus/issues/314) で 5 分から延長） |
| `QUEUE_TASK_TIMEOUT` | キュー投入タスクのタイムアウト（**default 540000 = 9 分**、同 [#314](https://github.com/gamasenninn/tealus/issues/314)） |
| `CC_ACK_TTL_MS` | `@cc-<project>` 投入時に出る受付エコーの表示時間（default 5000 = 5 秒、v0.6.0 [#335](https://github.com/gamasenninn/tealus/issues/335)） |
| `OPENAI_IMAGE_MODEL` | 画像生成モデルの上書き（default `gpt-image-1`、v0.5.0 [#313](https://github.com/gamasenninn/tealus/issues/313)） |

!!! danger "★ 以前このページは逆のことを書いていました"
    「`TEALUS_USER_ID` / `TEALUS_PASSWORD` を推奨、旧名 `TEALUS_BOT_ID` / `TEALUS_BOT_PASS` は fallback」と書いていましたが、**誤りです**。

    **agent-server が読むのは `TEALUS_BOT_ID` / `TEALUS_BOT_PASS` だけ**で、`TEALUS_USER_ID` / `TEALUS_PASSWORD` を書いても読まれません（`agent-server/src/config.mts`）。表のとおりに設定すると **bot がログインできず、エージェントが黙って応答しない**状態になります。

    紛らわしいのは、agent-server が**子プロセス（Deep / Light v2 の CLI や MCP の道具）へ渡すときに `TEALUS_USER_ID` / `TEALUS_PASSWORD` という名前に置き換えている**ことです。そちらは MCP クライアント側の名前で、**`agent-server/.env` に書く名前とは別物**です。

## 7. 本番デプロイ

### Nginx設定例

PWA・ダッシュボード・メディアは **server（:3000）が配信**します。agent-server・rtc-server へも server が中継するので、Nginx からの転送先は **すべて :3000** です。

```nginx
server {
    listen 443 ssl http2;
    server_name tealus.example.com;

    ssl_certificate     /etc/letsencrypt/live/tealus.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tealus.example.com/privkey.pem;

    # PWA（client/dist は server が配信する）
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # REST API
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Socket.IO（WebSocket upgrade 必須）
    location /socket.io/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    # cc-queue のストリーム — 別マシンの Claude Code をつなぐ場合だけ必要（#214）
    # つなぎっぱなしの NDJSON なので buffering off と長い read timeout が要る
    location /agent-api/cc-queue/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_read_timeout 3600s;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # agent-server（server が中継）
    location /agent-api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # rtc-server（通話・トランシーバー、WebSocket upgrade 必須）
    location /rtc/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    # 管理者ダッシュボード
    location /system/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }

    # メディア（server 経由で配信する）
    location /media/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
    }
}

server {
    listen 80;
    server_name tealus.example.com;
    return 301 https://$host$request_uri;
}
```

!!! danger "★ `/media/` を Nginx から直接配信（`alias`）しないでください"
    以前この例は `/media/` をディスクから直接配信していました。v0.11.0 から、**削除した投稿・部屋の添付は server が 404 を返します**（[#601](https://github.com/gamasenninn/tealus/issues/601) [#602](https://github.com/gamasenninn/tealus/issues/602)）。Nginx が直接配ると**この判定を通らず、消したはずの添付がリンクで開けてしまいます**。

!!! warning "旧版の例にあった不足"
    以前の例には `/agent-api/`（cc-queue の `proxy_buffering off` を含む）・`/rtc/`・`/system/` がありませんでした。そのとおりに設定すると、ダッシュボード・通話・別マシンの Claude Code 連携が動きません。プロキシの内側で動かすときは [`TRUST_PROXY`](#all-env) も設定してください。

### Cloudflare + NAS構成（参考）

NAS 上で運用する場合、所有ドメイン（例: `your-domain.example`）を Cloudflare 経由で NAS に接続する構成が便利です。

| サブドメイン例 | 用途 | 転送先 |
|---|---|---|
| `app.your-domain.example` | Tealus PWA | localhost:3000 |
| （不要） | ダッシュボード | 本番は server が `/system` で配信（`app.…/system`）。`localhost:5174` は開発用の dev server |
| `api.your-domain.example` | Agent Server | localhost:4000 |

Cloudflare Free プランでも SSL 自動発行・基本的な DDoS 緩和・CDN・オリジン IP 隠蔽が利用できますが、**業務利用では帯域・WAF ルール・Bot 対策など要件に応じて Pro / Business プランを検討**してください。

!!! warning "WebRTCポート"
    音声・ビデオ通話（mediasoup）を使用する場合、UDPポート **10000〜10100** をルーターで開放する必要があります。
