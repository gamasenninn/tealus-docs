# Bot API / Webhook

外部システムからTealusにメッセージを送受信するためのAPIです。

## Bot API

### 認証

Bot用ユーザーを作成し、`/api/auth/login` でJWTを取得します。

```bash
# JWTを取得
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"employee_id":"BOT001","password":"bot_password"}' \
  | jq -r '.token')
```

!!! note
    Botユーザーは `users.is_bot = true` に設定してください。これによりBotからのメッセージがWebhookで無限ループしなくなります。

### テキストメッセージ送信

```bash
curl -X POST http://localhost:3000/api/bot/push \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "room_id": "ルームUUID",
    "content": "Hello from Bot!"
  }'
```

Bot APIを使うと、Socket.IOブロードキャストが自動的に実行され、メッセージがリアルタイムで全参加者に配信されます。

!!! warning
    REST APIで直接DBに書き込んでもSocket.IOブロードキャストは走りません。必ずBot APIエンドポイントを使用してください。

### 画像送信

```bash
curl -X POST http://localhost:3000/api/bot/push-image \
  -H "Authorization: Bearer $TOKEN" \
  -F "room_id=ルームUUID" \
  -F "image=@/path/to/image.jpg"
```

### 音声送信

```bash
curl -X POST http://localhost:3000/api/bot/voice \
  -H "Authorization: Bearer $TOKEN" \
  -F "room_id=ルームUUID" \
  -F "voice=@/path/to/audio.wav"
```

音声送信時は自動的にWhisper文字起こし + AI整形が実行されます。

### メッセージ取得（ポーリング）

Webhookが使えない環境では、ポーリングでメッセージを取得できます。

```bash
curl -X GET "http://localhost:3000/api/bot/messages?room_id=UUID&since=2026-04-23T00:00:00Z" \
  -H "Authorization: Bearer $TOKEN"
```

## Webhook

### 概要

Tealusのイベントを外部HTTPエンドポイントに通知する機能です。

### イベント種別

| イベント | 説明 |
|---|---|
| `message.created` | メッセージ送信 |
| `message.deleted` | メッセージ削除 |
| `member.joined` | メンバー参加 |
| `member.left` | メンバー退会 |
| `reaction.added` | リアクション追加 |

### Webhook登録

管理画面の「Webhook」タブから登録するか、APIで登録します。`room_id` を `NULL` にすると全ルームのイベントを受信します。

### 署名検証

ペイロードは HMAC-SHA256 で署名されます。

```
X-Tealus-Signature: sha256=HMAC(secret, body)
```

受信側で署名を検証するサンプル（Node.js）:

```javascript
const crypto = require('crypto');

function verifySignature(body, signature, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}
```

### リトライポリシー

配信失敗時は指数バックオフで3回リトライ:

1. 5秒後
2. 15秒後
3. 45秒後
