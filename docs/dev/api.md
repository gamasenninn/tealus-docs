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
| POST | `/api/messages/:id/transcription/retranscribe` | 文字起こし retry（v0.2.2〜、新 version 作成） | 必要 |

`retranscribe` は文字起こしが失敗した（`status=error`）場合や結果が不適切な場合の手動 retry 用です（[#216](https://github.com/gamasenninn/tealus/issues/216)）。新しい version の文字起こしを生成し、Socket.IO 経由でリアルタイムに UI が更新されます。既存の編集機構（PUT route）と同じく version 履歴は保持されます。

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
| POST | `/api/bot/voice` | 音声送信（自動文字起こし、default `gpt-4o-transcribe`） |
| POST | `/api/bot/media` | メディア送信 |
| POST | `/api/bot/status` | ステータス送信 |
| GET | `/api/bot/messages` | メッセージ取得（ポーリング、v0.7.0〜 `include_transcription` / `include_raw` query parameter で文字起こし verbosity 制御） |
| GET | `/api/bot/messages/:id/media` | メッセージのメディア取得（画像/音声、AI 直接視認用、v0.1.x） |
| GET | `/api/bot/search` | メッセージ全文検索（v0.1.x） |
| GET | `/api/bot/unread` | 未読メッセージ取得 |
| POST | `/api/bot/mark-read` | 既読マーク |
| PATCH | `/api/bot/messages/:id/tags/:tag_name/done` | タグの完了状態（is_done）を更新（v0.1.x） |
| GET | `/api/bot/rooms` | ルーム一覧 |
| POST | `/api/bot/rooms` | ルーム作成 |

## Bot API 拡張（v0.1.x）

v0.1.x で追加された 3 つの Bot API endpoint の詳細です。**いずれも `server/src/routes/bot.js` に実装された通常の HTTP API** で、Bot 認証（JWT）で直接呼び出せます。[tealus-mcp](https://github.com/gamasenninn/tealus-mcp) の `search_messages` / `get_message_media` / `mark_tag_done` ツールは、これら HTTP API のラッパーとして AI エージェント向けに提供されているものです。

### `GET /api/bot/search` — メッセージ全文検索

メッセージ・音声文字起こしを横断的に検索します。`pg_trgm` GIN index と UNION クエリにより、3 文字以上のキーワードで 70-80x の高速化を実現しています（[#194](https://github.com/gamasenninn/tealus/issues/194)）。

**Query parameters**:

| パラメータ | 型 | 説明 |
|---|---|---|
| `q` | string | キーワード（ILIKE、3 文字以上推奨） |
| `room_id` | string | 単一ルームに絞り込み |
| `sender_id` | string | 発言者の user ID で絞り込み |
| `type` | enum | `text` / `image` / `voice` / `video` / `stamp` / `system` |
| `tag_names` | string | CSV、タグ AND 検索（例: `TODO,important`） |
| `is_done` | bool | TODO 完了状態（`tag_names` 指定時のみ有効） |
| `since` | ISO 8601 | 開始日時 |
| `until` | ISO 8601 | 終了日時 |
| `limit` | number | 1-50（デフォルト 10） |
| `offset` | number | デフォルト 0、`has_more=true` 時の続きは `next_offset` |

!!! warning "narrowing filter 必須"
    `q` / `room_id` / `sender_id` / `since` / `tag_names` / `type` のうち**最低 1 つ**を指定する必要があります。open-ended な全 DB スキャンを防ぐためです。

**Response**:

```json
{
  "results": [
    {
      "message_id": "...",
      "room_id": "...",
      "room_name": "業務メモ",
      "sender_id": "...",
      "sender_display_name": "田中太郎",
      "type": "text",
      "created_at": "2026-04-28T05:54:16.861Z",
      "snippet": "..."
    }
  ],
  "has_more": false,
  "next_offset": null
}
```

**curl サンプル**:

```bash
curl -G "http://localhost:3000/api/bot/search" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "q=TTS" \
  --data-urlencode "since=2026-04-26T00:00:00Z" \
  --data-urlencode "limit=10"
```

### `PATCH /api/bot/messages/:id/tags/:tag_name/done` — TODO 完了マーク

メッセージに付与されたタグの `is_done` 状態を切り替えます。voice メモから AI が抽出した TODO の完了化などに利用します（[#197](https://github.com/gamasenninn/tealus/issues/197)）。

**Body**:

```json
{ "is_done": true }
```

**Response**:

```json
{ "success": true, "updated": 1 }
```

**curl サンプル**:

```bash
curl -X PATCH "http://localhost:3000/api/bot/messages/$MSG_ID/tags/TODO/done" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"is_done": true}'
```

### `GET /api/bot/messages/:id/media` — メッセージのメディア取得

画像メッセージ・音声メッセージのメディアデータを取得します。AI エージェントが画像を**直接視認**するためのエンドポイントです（[#185](https://github.com/gamasenninn/tealus/issues/185)）。

| メディア種別 | レスポンス |
|---|---|
| 画像 | binary stream（Content-Type: `image/*`）。MCP ツール（`get_message_media`）経由では MCP 仕様の image content に変換され、AI が画像内容を直接認識可能 |
| 音声 | 文字起こしテキストを優先返却（`voice_transcriptions.formatted_text`）。生 wav が必要な場合は `?raw=1` |

**curl サンプル**:

```bash
curl "http://localhost:3000/api/bot/messages/$MSG_ID/media" \
  -H "Authorization: Bearer $TOKEN" \
  -o output.jpg
```

!!! info "MCP ツールとの関係"
    [Tealus MCP](mcp.md) の `get_message_media` ツールは、本 HTTP API のラッパーです。AI エージェントから利用する場合は MCP ツール経由が標準で、MCP 側で画像/音声の種別判定と MCP 仕様の返却形式（image content / 文字起こしテキスト）への変換が行われます。HTTP API として直接呼ぶことも可能です。

### voice transcription verbosity（v0.7.0）

`GET /api/bot/messages` および MCP `get_messages` で、voice メッセージの文字起こしを 3 段階で出し分けできます（[tealus #219](https://github.com/gamasenninn/tealus/issues/219) / [tealus-mcp #1](https://github.com/gamasenninn/tealus-mcp/issues/1)）。

**Query parameters**:

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `include_transcription` | bool | `true` | `false` で transcription 全体を `{id, status, version}` のみに圧縮（id-only、軽量モード） |
| `include_raw` | bool | `false` | `true` で `raw_text`（生 wav 文字起こし）も含める |

**Response の `transcription` field 3 段階**:

| 組合せ | `transcription` |
|---|---|
| `include_transcription=false` | `{id, status, version}` |
| **default**（`include_raw=false`） | `{id, status, version, formatted_text}` |
| `include_raw=true` | `{id, status, version, formatted_text, raw_text}` |

!!! note "後方互換と improvement"
    旧形式 `{raw_text, formatted_text, status}` から default 振る舞いが変わります（`raw_text` 省略 + `id` / `version` 追加）。本体側 consumer（`agent-server/src/media/messageAdapter.js` 等）は `formatted_text || raw_text` fallback chain で機能するため実害なし、むしろ整形空 voice で raw garbage を LLM に渡さなくなる improvement として動きます。

## ヘルスチェック

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/health` | ヘルスチェック | 不要 |
