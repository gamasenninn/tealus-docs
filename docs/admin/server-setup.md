# サーバー構築

## 前提条件

- **Node.js 20 以上必須**（v0.2.1〜、`engine-strict=true` でハードフェイル）
- Docker / Docker Compose
- Git

!!! warning "Node 18 では install 失敗"
    v0.2.1（[#210](https://github.com/gamasenninn/tealus/issues/210)）から全 6 package.json に `engines.node: ">=20.0.0"` + ルート `.npmrc` に `engine-strict=true` が設定されています。**Node 18 環境では `npm install` 自体が `EBADENGINE` で hard fail** します（原因: Node 18 に `File` global が未実装で `undici` v6+ が crash）。Node 20 系のインストールには nvm（推奨）または NodeSource パッケージを利用してください。

## 1. リポジトリのクローン

```bash
git clone https://github.com/gamasenninn/tealus.git
cd tealus
```

## 2. Docker 起動（開発用: DB + Redis のみ）

```bash
docker-compose up -d
```

以下のサービスが起動します。アプリケーション本体（server / agent-server）はホスト側で `npm run dev` する開発フロー想定です。

| サービス | ポート | 用途 |
|---|---|---|
| PostgreSQL | 5432 | 開発用DB |
| PostgreSQL | 5433 | テスト用DB |
| Redis | 6379 | セッション・在席状態管理 |

## 2.5 Docker フルデプロイ（本番想定）

v0.1.x（[#188](https://github.com/gamasenninn/tealus/issues/188)）から、server / agent-server / rtc-server を含む全サービスを Docker で一括起動できるようになりました。3 構成から運用形態に合わせて選択します。

### 3 つの構成

| 構成 | 起動コマンド | 含まれるサービス | 対応OS |
|---|---|---|---|
| 開発（DB + Redis のみ） | `docker-compose up -d` | postgres / postgres-test / redis | Mac / Windows / Linux |
| **フル**（server + agent 含む） | `docker-compose -f docker-compose.full.yml up -d` | 上記 + server:3000 + agent-server:4000 | Mac / Windows / Linux |
| + RTC（通話・legacy TTS） | `docker-compose -f docker-compose.full.yml -f docker-compose.rtc.yml up -d` | 上記 + rtc-server:3100 | **Linux のみ** |

!!! info "TTS のみなら rtc-server は不要"
    v0.1.x の default TTS 配信は Socket.IO blob 経由のため、TTS だけ使うなら **+RTC 構成は不要**です。WebRTC 通話機能を使う場合、または `TTS_BROADCAST_MEDIASOUP=true` で legacy mediasoup TTS を使う場合のみ rtc-server が必要になります。

### 各 Dockerfile の概要

| サービス | ベースイメージ | 構成 | 主な特徴 |
|---|---|---|---|
| server | `node:20-alpine` | 3 段階 multi-stage（client build → dashboard build → runtime） | client/dist と dashboard/dist を同梱、最終 ~312MB |
| agent-server | `node:20-alpine` | 単段ビルド | webhook + AI agent のみで軽量、最終 ~261MB |
| rtc-server | `node:20-bookworm-slim` | 2 段階 multi-stage（builder bookworm-slim → runtime bookworm-slim） | mediasoup の native worker が glibc 必須のため bookworm（Debian）ベース、最終 ~307MB |

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

### 全環境変数一覧

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
| `VAPID_SUBJECT` | `mailto:admin@example.com` | Web Push送信者メール（運用ドメインのアドレスに） |
| `OPENAI_API_KEY` | — | OpenAI APIキー |
| `MEDIA_ROOT` | ../../media | アップロードファイル保存先 |
| `AGENT_PORT` | 4000 | エージェントサーバーポート |
| `RTC_PORT` | 3100 | mediasoupシグナリングポート（rtc-server を使う場合） |
| `TTS_BROADCAST_MEDIASOUP` | `false` | TTS を mediasoup 経由で配信（legacy）。`true` で v0.1.0 以前の rtc-server 経由配信に切替（[#189](https://github.com/gamasenninn/tealus/issues/189)） |
| `LOG_LEVEL` | debug（開発） / info（本番） | ログレベル |
| `NODE_ENV` | — | `production` 設定時にCORSを制限 |
| `VITE_ALLOWED_HOSTS` | — | 外部ホスト経由で vite dev server に access する場合に許可ホストを指定（v0.2.0〜の注意点） |

### DBマイグレーション

```bash
npm run migrate
```

マイグレーションは `server/src/db/migrations/` 内の連番SQLファイル（001〜021）が順に実行されます。`021` で全文検索高速化用の `pg_trgm` 拡張と GIN index が追加されます（v0.1.x、[#194](https://github.com/gamasenninn/tealus/issues/194)）。

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
    v0.2.1（[#211](https://github.com/gamasenninn/tealus/issues/211)）から、**最初の非 Bot ユーザー**（`is_bot=false` の COUNT が 0 の状態で `register` を呼んだユーザー）が自動的に `admin` role で作成されます。Mattermost / Rocket.Chat / GitLab 標準の OSS first-user-admin パターンです。2 人目以降は通常 `user` role で作成され、必要に応じて管理者ダッシュボードから昇格させてください。

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
| `TEALUS_USER_ID` | BotユーザーのユーザーID |
| `TEALUS_PASSWORD` | Botユーザーのパスワード |
| `OPENAI_API_KEY` | OpenAI APIキー |
| `AGENT_WORKSPACE_ROOT` | ワークスペースのパス |
| `AIVIS_API_KEY` | Aivis Cloud API キー（TTS 音声合成用） |
| `AIVIS_MODEL_UUID` | デフォルト音声モデル UUID（デフォルト: 凛音エル） |
| `TTS_ENABLED` | TTS 読み上げの有効/無効（`"false"` で無効化、デフォルト: 有効） |
| `TTS_MAX_LENGTH` | 読み上げ最大文字数（デフォルト: 500） |
| `WHISPER_MODEL` | 音声文字起こしモデル（v0.2.2〜、default `gpt-4o-transcribe`、旧モデルに戻すなら `whisper-1`、半額の選択肢として `gpt-4o-mini-transcribe`） |

!!! note "旧環境変数名との互換"
    旧名 `TEALUS_BOT_ID` / `TEALUS_BOT_PASS` も内部で fallback されますが、新規セットアップでは `TEALUS_USER_ID` / `TEALUS_PASSWORD` の使用を推奨します（v0.1.x、Tealus MCP との表記統一）。

## 7. 本番デプロイ

### Nginx設定例

```nginx
server {
    listen 80;
    server_name tealus.example.com;

    # React PWA（ビルド済みの静的ファイル）
    location / {
        root /var/tealus/client/dist;
        try_files $uri $uri/ /index.html;
    }

    # REST API
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket（Socket.IO）
    location /socket.io/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # メディアファイル配信
    location /media/ {
        alias /var/tealus/media/;
        expires 30d;
    }
}
```

### Cloudflare + NAS構成（参考）

NAS 上で運用する場合、所有ドメイン（例: `your-domain.example`）を Cloudflare 経由で NAS に接続する構成が便利です。

| サブドメイン例 | 用途 | 転送先 |
|---|---|---|
| `app.your-domain.example` | Tealus PWA | localhost:3000 |
| `system.your-domain.example` | ダッシュボード | localhost:5174 |
| `api.your-domain.example` | Agent Server | localhost:4000 |

Cloudflare Free プランでも SSL 自動発行・基本的な DDoS 緩和・CDN・オリジン IP 隠蔽が利用できますが、**業務利用では帯域・WAF ルール・Bot 対策など要件に応じて Pro / Business プランを検討**してください。

!!! warning "WebRTCポート"
    音声・ビデオ通話（mediasoup）を使用する場合、UDPポート **10000〜10100** をルーターで開放する必要があります。
