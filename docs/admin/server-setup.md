# サーバー構築

## 前提条件

- Node.js 18+
- Docker / Docker Compose
- Git

## 1. リポジトリのクローン

```bash
git clone https://github.com/gamasenninn/tealus.git
cd tealus
```

## 2. Docker起動（PostgreSQL + Redis）

```bash
docker-compose up -d
```

以下のサービスが起動します。

| サービス | ポート | 用途 |
|---|---|---|
| PostgreSQL | 5432 | 開発用DB |
| PostgreSQL | 5433 | テスト用DB |
| Redis | 6379 | セッション・在席状態管理 |

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

| 変数名 | デフォルト | 説明 |
|---|---|---|
| `PORT` | 3000 | サーバーポート |
| `DB_HOST` | localhost | PostgreSQLホスト |
| `DB_PORT` | 5432 | PostgreSQLポート |
| `DB_NAME` | tealus | データベース名 |
| `DB_USER` | tealus | データベースユーザー |
| `DB_PASSWORD` | tealus_dev | データベースパスワード |
| `JWT_SECRET` | tealus-dev-secret | JWT署名シークレット |
| `VAPID_PUBLIC_KEY` | — | Web Push公開鍵 |
| `VAPID_PRIVATE_KEY` | — | Web Push秘密鍵 |
| `VAPID_SUBJECT` | mailto:admin@tealus.local | Web Push送信者メール |
| `OPENAI_API_KEY` | — | OpenAI APIキー |
| `MEDIA_ROOT` | ../../media | アップロードファイル保存先 |
| `AGENT_PORT` | 4000 | エージェントサーバーポート |
| `RTC_PORT` | 3100 | mediasoupシグナリングポート |
| `LOG_LEVEL` | debug（開発） / info（本番） | ログレベル |
| `NODE_ENV` | — | `production` 設定時にCORSを制限 |

### DBマイグレーション

```bash
npm run migrate
```

マイグレーションは `server/src/db/migrations/` 内の連番SQLファイル（001〜018）が順に実行されます。

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

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"login_id":"admin","display_name":"管理者","password":"password123"}'
```

!!! warning
    初回に登録したユーザーを管理者にするには、データベースで直接 `role` を `admin` に設定する必要があります。

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
| `TEALUS_BOT_ID` | BotユーザーのユーザーID |
| `TEALUS_BOT_PASS` | Botユーザーのパスワード |
| `OPENAI_API_KEY` | OpenAI APIキー |
| `AGENT_WORKSPACE_ROOT` | ワークスペースのパス |
| `AIVIS_API_KEY` | Aivis Cloud API キー（TTS 音声合成用） |
| `AIVIS_MODEL_UUID` | デフォルト音声モデル UUID（デフォルト: 凛音エル） |
| `TTS_ENABLED` | TTS 読み上げの有効/無効（`"false"` で無効化、デフォルト: 有効） |
| `TTS_MAX_LENGTH` | 読み上げ最大文字数（デフォルト: 500） |

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

### Cloudflare + NAS構成

`tealus.dev` ドメインをCloudflare経由でNASに接続する構成が利用可能です。

| サブドメイン | 用途 | 転送先 |
|---|---|---|
| `app.tealus.dev` | Tealus PWA | localhost:3000 |
| `system.tealus.dev` | ダッシュボード | localhost:5174 |
| `api.tealus.dev` | Agent Server | localhost:4000 |

Cloudflare Free プランでSSL自動発行・DDoS保護・CDN・IP隠蔽がすべて無償で利用できます。

!!! warning "WebRTCポート"
    音声・ビデオ通話（mediasoup）を使用する場合、UDPポート **10000〜10100** をルーターで開放する必要があります。
