# Socket.IO イベント

リアルタイム通信に使用するSocket.IOイベントの一覧です。

## 認証

Socket.IO接続時に `handshake.auth.token` でJWTを検証します。管理者ユーザーは接続時に全ルームへ自動参加します（モニタリング用）。

## クライアント → サーバー

| イベント | ペイロード | 説明 |
|---|---|---|
| `room:join` | `roomId` | ルームに参加（メッセージ受信開始） |
| `room:leave` | `roomId` | ルームから退出（メッセージ受信停止） |
| `message:send` | `{ room_id, content, type, reply_to, forwarded_from? }` | メッセージ送信。★ **受け取りの返事（ack）**を返す: 保存できたら `{ ok: true, id }`、断ったら `{ ok: false, error: 理由 }`（[#593](https://github.com/gamasenninn/tealus/issues/593)）。画面は断られたら入力欄の文を残して理由を出す。ack 無しの emit（古い画面）も受け付ける |
| `message:read` | `{ room_id, message_ids }` | 既読マーク |
| `typing:start` | `room_id` | 入力中通知の開始 |
| `typing:stop` | `room_id` | 入力中通知の停止 |
| `room:viewing` | `{ room_id, viewing: boolean }` | その部屋を開いていて**画面が見えている**ことを知らせる。見ている人にはプッシュ通知を送らない（[#475](https://github.com/gamasenninn/tealus/issues/475)） |
| `call:start` | `{ roomId }` | 通話開始/参加 |
| `call:reject` | `{ roomId, callerId }` | 着信拒否 |
| `call:end` | `{ roomId }` | 通話終了 |
| `call:getStatus` | `{ roomId }` | 通話状態取得 |

## サーバー → クライアント

| イベント | 受信者 | 説明 |
|---|---|---|
| `message:new` | ルームメンバー全員 | 新着メッセージ通知 |
| `message:read` | その部屋の**他の接続全員**（`{ room_id, read_counts }`） | 既読数の更新 |
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
| `message:updated` | ルームメンバー全員 | メッセージの編集 |
| `message:published` | ルームメンバー全員 | お知らせへの公開・非公開（`{ message_id, is_published }`） |
| `message:tags` | ルームメンバー全員 | メッセージのタグの更新 |
| `room:tag_deleted` | ルームメンバー全員 | ルームのタグの削除 |
| `room:added` | 加わった本人（`user:{id}`） | 部屋に加わった（一覧に足す、[#486](https://github.com/gamasenninn/tealus/issues/486)） |
| `room:removed` | 外れた本人 | 部屋から外れた（一覧から消す、[#533](https://github.com/gamasenninn/tealus/issues/533)） |
| `room:updated` | ルームメンバー全員 | 部屋の情報が変わった（取り直す） |
| `user:updated` | 関係する利用者 | 表示名・アイコンの変更（[#534](https://github.com/gamasenninn/tealus/issues/534)） |
| `unread:changed` | 本人 | 未読数が変わった（一覧を取り直す） |
| `agent:status` | ルームメンバー全員 | エージェントの状態（考え中など、`POST /api/bot/status` から） |
| `tts:audio` | ルームメンバー全員 | 読み上げ音声の URL（`{ url }`。各端末が取りに行く） |
| `tts:speak` | ルームメンバー全員 | ブラウザ標準音声で読む文（`TTS_PROVIDER=browser`） |
| `stamp:generated` / `stamp:error` | 生成を頼んだ本人 | スタンプ生成の完了・失敗 |
| `capability:changed` | 全接続 | 通話・トランシーバーが使えるか（rtc-server の到達性）が変わった |

`message:new` などで届くメッセージには、転送されたものなら `forwarded_from`（転送元の要約。元が削除されていれば「元メッセージは削除されました」の扱い）が含まれます。
