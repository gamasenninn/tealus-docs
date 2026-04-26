# API リファレンス

すべてのAPIエンドポイントの一覧です。認証が必要なエンドポイントは `Authorization: Bearer <JWT>` ヘッダが必要です。

## 共通仕様

### エラーレスポンス

エラーレスポンスは以下の JSON 形式で返されます。

```json
{ "error": "エラーメッセージ" }
```

### ステータスコード

| コード | 用途 |
|---|---|
| 200 | 成功 |
| 201 | リソース作成成功 |
| 400 | バリデーションエラー（必須フィールド欠如、不正な値） |
| 401 | 認証エラー（トークン未送信・無効・パスワード不一致） |
| 403 | 権限エラー（ルームアクセス権なし・admin権限不足） |
| 404 | リソースが見つからない |
| 409 | 重複（既にメンバーに存在等） |
| 500 | サーバー内部エラー |

### リクエスト/レスポンス例

#### ログイン

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"login_id":"admin","password":"<your_password>"}'
```

```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "login_id": "admin",
    "display_name": "管理者",
    "role": "admin"
  }
}
```

#### メッセージ送信

```bash
curl -X POST http://localhost:3000/api/rooms/{roomId}/messages \
  -H "Authorization: Bearer <JWT>" \
  -H "Content-Type: application/json" \
  -d '{"content":"こんにちは","type":"text"}'
```

#### メッセージ取得（ページネーション）

```bash
curl "http://localhost:3000/api/rooms/{roomId}/messages?limit=50&before={messageId}" \
  -H "Authorization: Bearer <JWT>"
```

## 認証

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/auth/register` | ユーザー登録 | 不要 |
| POST | `/api/auth/login` | ログイン（JWT発行） | 不要 |
| GET | `/api/auth/me` | 現在ユーザー取得 | 必要 |
| PUT | `/api/auth/profile` | プロフィール更新（表示名・ステータス） | 必要 |
| POST | `/api/auth/avatar` | プロフィール画像アップロード | 必要 |
| PUT | `/api/auth/password` | パスワード変更 | 必要 |

## ユーザー

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/users` | ユーザー一覧 | 必要 |
| GET | `/api/users/online` | オンラインユーザーID一覧 | 必要 |

## ルーム

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/rooms` | ルーム一覧（未読数・メンバー数付き） | 必要 |
| POST | `/api/rooms` | グループ作成 | 必要 |
| POST | `/api/rooms/direct` | 1対1ルーム作成 | 必要 |
| GET | `/api/rooms/:id` | ルーム詳細 | 必要 |
| PUT | `/api/rooms/:id` | グループ名変更 | 必要 |
| POST | `/api/rooms/:id/icon` | グループアイコンアップロード | 必要 |
| GET | `/api/rooms/portal-links` | ポータルリンク一覧 | 必要 |
| GET | `/api/rooms/announcements` | お知らせ一覧 | 必要 |

## メッセージ

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/rooms/:id/messages` | メッセージ履歴（ページネーション） | 必要 |
| POST | `/api/rooms/:id/messages` | メッセージ送信 | 必要 |
| PUT | `/api/rooms/:id/messages/:msgId` | メッセージ編集 | 必要 |
| GET | `/api/rooms/:id/messages/:msgId/edits` | 編集履歴取得 | 必要 |
| DELETE | `/api/rooms/:id/messages/:msgId` | メッセージ削除（論理削除） | 必要 |
| POST | `/api/rooms/:id/messages/:msgId/reactions` | 絵文字リアクション（トグル） | 必要 |
| PATCH | `/api/rooms/:id/messages/:msgId/publish` | メッセージ公開/非公開 | 必要 |

## メディア・音声

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/rooms/:id/media` | ファイルアップロード（画像・動画・ファイル） | 必要 |
| GET | `/api/rooms/:id/media/gallery` | メディアギャラリー | 必要 |
| POST | `/api/rooms/:id/voice` | 音声メッセージアップロード（自動文字起こし） | 必要 |

## 文字起こし

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| PUT | `/api/messages/:id/transcription` | 文字起こしテキスト編集 | 必要 |
| GET | `/api/messages/:id/transcription/history` | 編集履歴取得 | 必要 |

## 既読

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/rooms/:id/read` | 既読マーク | 必要 |
| POST | `/api/rooms/:id/read/all` | すべて既読 | 必要 |

## メンバー管理

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/rooms/:id/members` | メンバー追加 | 必要 |
| DELETE | `/api/rooms/:id/members/me` | 自分が退会 | 必要 |
| DELETE | `/api/rooms/:id/members/:userId` | メンバー除外 | 必要 |
| PUT | `/api/rooms/:id/members/:userId/role` | グループ管理者変更 | 必要 |

## 検索

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/search` | メッセージ検索 | 必要 |

## タグ

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| — | `/api/tags` | グローバルタグ操作 | 必要 |
| — | `/api/rooms/:id/tags` | ルームタグ管理 | 必要 |
| — | `/api/messages/:id/tags` | メッセージタグ付け | 必要 |

## スタンプ

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/stamps/generate` | AIスタンプパック生成 | 必要 |
| GET | `/api/stamps/packs` | スタンプパック一覧 | 必要 |
| GET | `/api/stamps/packs/:id` | パック詳細 | 必要 |
| PUT | `/api/stamps/packs/:id` | パック更新 | 必要 |
| DELETE | `/api/stamps/packs/:id` | パック削除 | 必要 |
| DELETE | `/api/stamps/:id` | スタンプ削除 | 必要 |

## プッシュ通知

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/push/subscribe` | Push通知購読登録 | 必要 |
| DELETE | `/api/push/subscribe` | Push通知購読解除 | 必要 |

## 管理者API

管理者ロール（`admin`）が必要です。

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/admin/users` | ユーザー一覧 |
| POST | `/api/admin/users` | ユーザー作成 |
| PUT | `/api/admin/users/:id` | ユーザー編集 |
| PATCH | `/api/admin/users/:id/status` | ユーザー有効化/無効化 |
| — | `/api/admin/portal-links` | ポータルリンクCRUD |
| — | `/api/admin/webhooks` | Webhook CRUD |
| POST | `/api/admin/webhooks/:id/test` | Webhookテスト送信 |
| GET | `/api/admin/agent-stats` | エージェント統計 |
| GET | `/api/admin/agent-logs` | エージェント応答ログ |

## Bot API

Bot用ユーザーとしてJWT認証が必要です。

| メソッド | パス | 説明 |
|---|---|---|
| POST | `/api/bot/push` | テキストメッセージ送信 |
| POST | `/api/bot/push-image` | 画像送信 |
| POST | `/api/bot/voice` | 音声送信（Whisper自動文字起こし） |
| POST | `/api/bot/media` | メディア送信 |
| POST | `/api/bot/status` | ステータス送信 |
| GET | `/api/bot/messages` | メッセージ取得（ポーリング） |
| GET | `/api/bot/unread` | 未読メッセージ取得 |
| POST | `/api/bot/mark-read` | 既読マーク |
| GET | `/api/bot/rooms` | ルーム一覧 |
| POST | `/api/bot/rooms` | ルーム作成 |

## ヘルスチェック

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/health` | ヘルスチェック | 不要 |
