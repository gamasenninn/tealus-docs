# アーキテクチャ

## システム全体構成

```
┌─────────────────────────────────────────────────┐
│                   クライアント                     │
│         React 19 + Vite + PWA + Zustand          │
│              http://localhost:5173                │
└───────────────┬─────────────────┬────────────────┘
                │ REST API        │ WebSocket
                │ /api/*          │ /socket.io
                ▼                 ▼
┌─────────────────────────────────────────────────┐
│                 Tealus Server                    │
│          Node.js + Express + Socket.IO           │
│              http://localhost:3000                │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │ Routes   │ │ Socket   │ │ Middleware       │ │
│  │ (REST)   │ │ Handlers │ │ (Auth/Upload)    │ │
│  └────┬─────┘ └────┬─────┘ └──────────────────┘ │
│       │             │                            │
│  ┌────▼─────────────▼────┐  ┌─────────────────┐ │
│  │     PostgreSQL 16     │  │    Redis 7       │ │
│  │      (RLS有効)        │  │  (セッション/    │ │
│  │     :5432             │  │   在席管理)      │ │
│  └───────────────────────┘  └─────────────────┘ │
└──────────┬──────────────────────┬────────────────┘
           │ Webhook              │ Proxy
           ▼                      ▼
┌──────────────────┐   ┌──────────────────┐
│  Agent Server    │   │  RTC Server      │
│  (AI エージェント) │   │  (mediasoup SFU) │
│  :4000           │   │  :3100           │
└──────────────────┘   └──────────────────┘
```

## コンポーネント構成

### フロントエンド（client/）

| ディレクトリ | 役割 |
|---|---|
| `components/auth/` | ログイン画面 |
| `components/home/` | ホーム画面 |
| `components/room-list/` | トーク一覧・ルーム作成 |
| `components/chat/` | チャット本体（吹き出し・入力・設定・メンバー一覧） |
| `components/call/` | 通話UI（mediasoup） |
| `components/media/` | 画像グリッド・ビューア・ギャラリー |
| `components/admin/` | 管理者ダッシュボード |
| `components/multi/` | マルチトーク（複数ルーム並列） |
| `components/search/` | 全文検索 |
| `components/stamp/` | AIスタンプ生成・送信 |
| `components/share/` | PWA Share Target |
| `hooks/` | カスタムフック（Socket同期・通話・スクロール等） |
| `stores/` | Zustand状態管理（auth・message・room） |
| `services/` | API/Socket.IO/Pushクライアント |

### バックエンド（server/）

| ディレクトリ | 役割 |
|---|---|
| `routes/` | REST APIルート定義（15ファイル） |
| `socket/handlers/` | Socket.IOイベントハンドラ（message・read・typing・call） |
| `middleware/` | JWT認証・ファイルアップロード |
| `services/` | プッシュ通知・サムネイル生成等のビジネスロジック |
| `db/` | DB接続・マイグレーション（001〜018） |
| `utils/` | ユーティリティ |

### AIエージェント（agent-server/）

| ディレクトリ | 役割 |
|---|---|
| `agents/` | Light/Deep エージェント実装 |
| `webhook/` | Webhookディスパッチャー・ハンドラ |
| `mcp/` | MCPマネージャー |
| `context/` | セッション・設定マネージャー |
| `memory/` | ファイルベースメモリ |

## データベース設計

### コアテーブル

| テーブル | 役割 |
|---|---|
| `users` | ユーザー（ユーザーID・表示名・アバター・ロール） |
| `rooms` | トークルーム（direct/group） |
| `room_members` | ルーム参加者（ロール: admin/member） |
| `messages` | メッセージ（テキスト・画像・動画・音声・スタンプ・システム） |
| `message_media` | 添付ファイル（ファイルパス・MIME・サムネイル） |
| `room_read_cursors` | 既読カーソル（一覧用の未読数管理） |
| `message_reads` | 既読記録（チャット画面の既読数表示） |
| `push_subscriptions` | プッシュ通知購読 |

### 拡張テーブル

| テーブル | 役割 |
|---|---|
| `voice_transcriptions` | 音声文字起こし（Whisper連携） |
| `tags` / `message_tags` | タグ・TODO機能 |
| `stamp_packs` / `stamps` | AIスタンプ |
| `webhooks` | Webhook設定 |
| `agent_contexts` | AIエージェントセッション |

### Row Level Security（RLS）

`messages`、`room_members`、`message_reads` テーブルにRLSが有効です。`current_setting('app.current_user_id', true)` でユーザーを識別し、自分が参加しているルームのデータのみアクセス可能です。

## 認証フロー

```
POST /api/auth/login { login_id, password }
    ↓ bcryptで検証
JWT発行（ペイロード: user_id, role）
    ↓
Authorization: Bearer <token>
    ↓ middleware/auth.js で検証
req.user にユーザー情報を設定
```

Socket.IO接続時も `handshake.auth.token` でJWTを検証します。

## TTS 音声合成パイプライン（PlainTransport）

AIエージェントの回答を音声で読み上げる際、mediasoup の **PlainTransport** を使って WebRTC スタックなしで音声を注入します。

### 処理フロー

```
agent-server: speakMessage(roomId, text)
  → テキスト前処理（Markdown除去・URL省略・500文字制限）
  → Aivis Cloud API (HTTPS POST) → WAV バイナリ
  → sendViaPlainTransport(wavFile, roomId)
      ├── WebSocket → rtc-server (localhost:3100)
      ├── join → createPlainTransport(comedia: true)
      ├── plainProduce(ssrc: 1111)
      ├── Consumer セットアップ待ち 2秒
      └── ffmpeg → Opus RTP → PlainTransport → Router
          → WebRTC Consumer → ブラウザで再生
```

### PlainTransport の仕組み

通常の WebRTC 通話では、ブラウザ間で DTLS/SRTP を使った暗号化通信を行います。一方 PlainTransport は **暗号化なしの RTP/RTCP を直接受け取る**トランスポートで、サーバーサイドの音声ソース（ffmpeg 等）から mediasoup に音声を注入する用途に適しています。

**comedia モード**: 送信元の IP:port を事前に指定する代わりに、最初の RTP パケットから自動検知します。ローカルホスト内の通信で利用しています。

### ffmpeg パラメータ

```bash
ffmpeg -re -i <wavFile> \
  -af adelay=300|300,apad=pad_dur=500ms \
  -c:a libopus -ac 2 -ar 48000 -b:a 32k \
  -f rtp -ssrc 1111 -payload_type 100 \
  rtp://127.0.0.1:<port>
```

| パラメータ | 値 | 説明 |
|---|---|---|
| `-re` | — | リアルタイム速度で送信 |
| `adelay=300\|300` | 300ms | 頭切れ防止の先頭無音 |
| `apad=pad_dur=500ms` | 500ms | 尻切れ防止の末尾無音 |
| `-c:a libopus` | Opus | コーデック |
| `-ar 48000` | 48kHz | サンプリングレート |
| `-b:a 32k` | 32kbps | ビットレート |
| `-ssrc 1111` | 固定値 | RTP ストリーム識別子 |
| `-payload_type 100` | 100 | mediasoup と合わせた PT 番号 |
