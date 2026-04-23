# Socket.IO イベント

リアルタイム通信に使用するSocket.IOイベントの一覧です。

## 認証

Socket.IO接続時に `handshake.auth.token` でJWTを検証します。管理者ユーザーは接続時に全ルームへ自動参加します（モニタリング用）。

## クライアント → サーバー

| イベント | ペイロード | 説明 |
|---|---|---|
| `room:join` | `roomId` | ルームに参加（メッセージ受信開始） |
| `room:leave` | `roomId` | ルームから退出（メッセージ受信停止） |
| `message:send` | `{ room_id, content, type, reply_to }` | メッセージ送信 |
| `message:read` | `{ room_id, message_ids }` | 既読マーク |
| `typing:start` | `room_id` | 入力中通知の開始 |
| `typing:stop` | `room_id` | 入力中通知の停止 |
| `call:start` | `{ roomId }` | 通話開始/参加 |
| `call:reject` | `{ roomId, callerId }` | 着信拒否 |
| `call:end` | `{ roomId }` | 通話終了 |
| `call:getStatus` | `{ roomId }` | 通話状態取得 |

## サーバー → クライアント

| イベント | 受信者 | 説明 |
|---|---|---|
| `message:new` | ルームメンバー全員 | 新着メッセージ通知 |
| `message:read` | メッセージ送信者 | 既読更新通知 |
| `message:deleted` | ルームメンバー全員 | メッセージ削除通知 |
| `message:reaction` | ルームメンバー全員 | リアクション更新 |
| `voice:status` | ルームメンバー全員 | 文字起こしステータス更新 |
| `voice:transcription` | ルームメンバー全員 | 文字起こし結果 |
| `link:preview` | ルームメンバー全員 | リンクプレビュー結果 |
| `typing:start` | ルームの他メンバー | 入力中表示 |
| `typing:stop` | ルームの他メンバー | 入力中表示の停止 |
| `user:online` | 全接続ユーザー | ユーザーオンライン通知 |
| `user:offline` | 全接続ユーザー | ユーザーオフライン通知 |
| `member:added` | ルームメンバー全員 | メンバー追加通知 |
| `member:removed` | ルームメンバー全員 | メンバー退会/除外通知 |
| `call:incoming` | 着信ユーザー | 通話着信通知（DM） |
| `call:rejected` | 発信者 | 通話拒否通知 |
| `call:ended` | ルームメンバー | 通話終了通知 |
| `call:status` | ルームメンバー | 通話状態（参加人数等） |
