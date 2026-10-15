# API リファレンス

すべてのAPIエンドポイントの一覧です。認証が必要なエンドポイントは `Authorization: Bearer <JWT>` ヘッダが必要です。

## 共通仕様

### エラーレスポンス

エラーレスポンスは以下の JSON 形式で返されます。

```json
{ "error": "エラーメッセージ" }
```

- ★ **想定外の失敗（500）は、決まった文だけを返します**（[#603](https://github.com/gamasenninn/tealus/issues/603)）。中身（スタック・本体の中のパス）は返さず、サーバーの記録にだけ残します。`NODE_ENV` に関係ありません。状態の番号を持つ失敗（本文が大きすぎる 413・JSON が壊れている 400 など）は番号を保ち、本文は「リクエストを受け付けられませんでした」です
- ★ **ID の形が正しくないと 400** です（[#543](https://github.com/gamasenninn/tealus/issues/543)）。以前は 500 になっていました。本文は「ID「…」の形が正しくありません。ID は省略せずに、36 文字のまま渡してください。」で、**途中で切った ID を渡した AI が自分で気づける**ようにしてあります

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
| POST | `/api/auth/register` | ユーザー登録（★ **人の利用者が 0 人のときだけ**。1 人でもいると 403、[#469](https://github.com/gamasenninn/tealus/issues/469)。2 人目以降は管理 API `/api/admin/users`） | 不要 |
| POST | `/api/auth/login` | ログイン（JWT発行） | 不要 |
| GET | `/api/auth/me` | 現在ユーザー取得 | 必要 |
| GET | `/api/auth/authz?room_id=` | この人は誰か・このルームでの役割（[#458](https://github.com/gamasenninn/tealus/issues/458)）。agent-server が**利用者自身の鍵で**聞き、設定を触らせてよいかを判断する。返すのは事実だけ | 必要 |
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
| PUT | `/api/rooms/:id` | ルーム設定の更新（名前・編集の決まり・会話モード・機械の投稿の通知など。`is_announcement` は admin のみ） | 必要 |
| PUT | `/api/rooms/:id/notification` | **自分の分だけ**このルームの通知を切り替え（`{ "muted": boolean }`、[#463](https://github.com/gamasenninn/tealus/issues/463)）。通話の着信は止めない | 必要 |
| DELETE | `/api/rooms/:id` | グループ削除。**作成者で、メンバーが自分だけのとき**。メッセージと添付も消える | 必要 |
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
| GET | `/api/rooms/:id/prompts/history?targets=アシスタント,cc-tealus&limit=30` | 自分がこのルームで送った「宛先 + 本文」の投稿を新しい順に（「最近の指示」パネル、[#354](https://github.com/gamasenninn/tealus/issues/354)） | 必要 |
| PATCH | `/api/rooms/:id/messages/:msgId/publish` | お知らせに公開/非公開（送信者か部屋の管理者） | 必要 |
| GET | `/api/rooms/:id/messages/days?month=YYYY-MM&tz_offset=分` | その月で投稿がある日（カレンダーの印、[#476](https://github.com/gamasenninn/tealus/issues/476)） | 必要 |
| GET | `/api/rooms/:id/messages/first-of-day?date=YYYY-MM-DD&tz_offset=分` | その日の最初の投稿（無ければ `message_id: null`） | 必要 |

## メディア・音声

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| POST | `/api/rooms/:id/media` | ファイルアップロード（画像・動画・ファイル） | 必要 |
| GET | `/api/rooms/:id/media/gallery` | メディアギャラリー | 必要 |
| POST | `/api/rooms/:id/media/forward` | 画像・動画・ファイルの投稿を別のルームへ転送（元のルームのメンバーであること。ファイルの実体は共有） | 必要 |
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
| DELETE | `/api/push/subscribe` | Push通知購読解除。★ **この口だけは、署名が正しければ期限切れのトークンでも受け付けます**（期限切れでログアウトするときに解除できず、宛先が残っていたため） | 必要 |

## その他

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/config` | 画面が使う設定（通話・トランシーバーが使えるか等。agent-server の公開設定も中継） | 不要 |
| GET | `/api/version` | 本体のビルド ID。画面は自分のビルド ID と突き合わせて「更新あり」を判断する | 不要 |
| POST | `/api/client-errors` | 画面で起きたエラーを本体の記録へ送る（[#525](https://github.com/gamasenninn/tealus/issues/525)） | 不要 |

## 配信 `/media`

アップロードしたファイルは `/media/...` で配信されます。★ **削除した投稿・削除した部屋の添付は 404** を返します（v0.11.0、[#601](https://github.com/gamasenninn/tealus/issues/601) [#602](https://github.com/gamasenninn/tealus/issues/602)）。リバースプロキシでディスクから直接配ると、この判定を通らなくなります（[サーバー構築 > Nginx設定例](../admin/server-setup.md)）。

## 管理者API

管理者ロール（`admin`）が必要です。

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/admin/users` | ユーザー一覧 |
| POST | `/api/admin/users` | ユーザー作成 |
| PUT | `/api/admin/users/:id` | ユーザー編集 |
| PATCH | `/api/admin/users/:id/status` | ユーザー有効化/無効化 |
| GET / POST / PUT / DELETE | `/api/admin/portal-links`（`/:id`） | ポータルリンク |
| GET / POST / PUT / DELETE | `/api/admin/webhooks`（`/:id`） | Webhook |
| POST | `/api/admin/webhooks/:id/test` | Webhookテスト送信 |
| GET | `/api/admin/rooms` | 全ルーム一覧 |
| GET | `/api/admin/access-log` | 最終投稿・最終閲覧の集計（[#301](https://github.com/gamasenninn/tealus/issues/301)） |
| GET | `/api/admin/agent-stats` | エージェント統計 |
| GET | `/api/admin/agent-logs` | エージェント応答ログ |
| GET | `/api/admin/agent-logs/:id/context` | 応答ログ 1 件の前後の文脈 |
| GET / POST | `/api/admin/dictionary/terms` | 業務辞書の語（`PATCH /:id`・`PATCH /:id/reading`・`POST /:id/reject`・`POST /:id/restore`） |
| GET / POST | `/api/admin/dictionary/aliases` | 別名の候補（`POST /:id/approve`・`POST /:id/reject`） |
| POST | `/api/admin/transcription/reload-vocab` | **voice transcription の業務語彙辞書 cache を手動 reset**（v0.3.0〜、[#287](https://github.com/gamasenninn/tealus/issues/287)、`transcription_guideline.json` 編集後の即時反映用、server restart 不要）。v0.4.0（[#296](https://github.com/gamasenninn/tealus/issues/296)）で **mtime 自動リロード** が default 動作になり、本 endpoint は手動 reset の fallback として残存 |

## Bot API

Bot用ユーザーとしてJWT認証が必要です。

| メソッド | パス | 説明 |
|---|---|---|
| POST | `/api/bot/push` | テキストメッセージ送信 |
| POST | `/api/bot/push-image` | 画像送信 |
| POST | `/api/bot/status` | ステータス送信（入力中のような表示。メッセージにはならない） |
| POST | `/api/bot/heartbeat` | **鼓動**（`{ name, info }`）。[見張り](../admin/automation.md) の `heartbeat` が見る（[#595](https://github.com/gamasenninn/tealus/issues/595)） |
| POST | `/api/bot/tts-speak` | 各端末のブラウザ標準音声で読ませる（`TTS_PROVIDER=browser` のとき agent-server が使う。DB に書かない） |
| POST | `/api/bot/tts-audio` | 合成済みの音声（`audio`）を受け取り、ルームに配信（メモリに 5 分保持） |
| GET | `/api/bot/tts-audio/:id` | 上の音声を取得（期限切れは 404） |
| GET | `/api/bot/messages` | メッセージ取得（ポーリング、v0.7.0〜 `include_transcription` / `include_raw` query parameter で文字起こし verbosity 制御） |
| GET | `/api/bot/messages/:id/media` | メッセージのメディア取得（画像/音声、AI 直接視認用、v0.1.x） |
| GET | `/api/bot/messages/:id/edit-history` | **メッセージ編集履歴の取得**（v0.3.0〜、text edits + voice / video transcription versions を統合返却、[#208](https://github.com/gamasenninn/tealus/issues/208) follow-up、organon daily cycle の edit history 観察用） |
| POST | `/api/bot/messages/:id/transcribe` | 動画/音声メッセージの文字起こし（v0.2.4〜、[#271](https://github.com/gamasenninn/tealus/issues/271)、tealus-mcp `transcribe_media` の base endpoint） |
| GET | `/api/bot/rooms/:id/membership` | **bot がルームメンバーか確認**（v0.3.0 後、[#282](https://github.com/gamasenninn/tealus/issues/282)、`%委譲%` 委譲権限 gate 用、未参加 room への委譲拒否で使用） |
| GET | `/api/bot/search` | メッセージ全文検索（v0.1.x） |
| GET | `/api/bot/tags` | タグ一覧を usage 順で返す discovery primitive（v0.2.3〜、[tealus #254](https://github.com/gamasenninn/tealus/issues/254)） |
| GET | `/api/bot/unread` | 未読メッセージ取得 |
| POST | `/api/bot/mark-read` | 既読マーク |
| PATCH | `/api/bot/messages/:id/tags/:tag_name/done` | タグの完了状態（is_done）を更新（v0.1.x） |
| GET | `/api/bot/rooms` | ルーム一覧 |
| POST | `/api/bot/rooms/:id/join` | bot が自分でルームに入る。★ **bot（`is_bot`）だけ**。人は 403（招待で入る） |
| POST | `/api/bot/push-file` | text 内容を file（.txt / .md 等）として投稿（v0.2.3〜、tealus-mcp v0.11.0 の `send_text_as_file` の wrapper） |

## Bot API 拡張（v0.1.x）

v0.1.x で追加された 3 つの Bot API endpoint の詳細です。**いずれも `server/src/routes/bot.mts` に実装された通常の HTTP API** で、Bot 認証（JWT）で直接呼び出せます。[tealus-mcp](https://github.com/gamasenninn/tealus-mcp) の `search_messages` / `get_message_media` / `mark_tag_done` ツールは、これら HTTP API のラッパーとして AI エージェント向けに提供されているものです。

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

### `GET /api/bot/tags` — タグ一覧 discovery（v0.2.3〜、[#254](https://github.com/gamasenninn/tealus/issues/254)）

bot がメンバーである全 room の tag 一覧を usage 順（利用回数 desc）で返します。LLM が `search_messages` で `tag_names` filter を組み立てる時、tag 名を知らないと「正解の名前を当てるゲーム」を強いられる構造的問題への対応として、**list / discovery primitive** を提供する endpoint です。

**Query parameters**:

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `limit` | number | 30 | 取得件数（max 100） |

**Response**:

```json
{
  "tags": [
    { "name": "TODO", "is_todo": true, "total_usage": 42 },
    { "name": "important", "is_todo": false, "total_usage": 18 }
  ]
}
```

- `is_todo`: `mark_tag_done` で完了状態を切り替えできる TODO 系 tag かどうか
- `total_usage`: bot が参加している全 room を横断した利用回数

`search_messages` の `tag_names` 指定の前段で discovery として呼び出すと、`is_done` filter と組み合わせて TODO 進捗管理 flow が成立します（tealus-mcp 側では `list_tags` ツールとしてラップされています）。

**curl サンプル**:

```bash
curl "http://localhost:3000/api/bot/tags?limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

### `GET /api/bot/messages/:id/media` — メッセージのメディア取得

画像メッセージ・音声メッセージのメディアデータを取得します。AI エージェントが画像を**直接視認**するためのエンドポイントです（[#185](https://github.com/gamasenninn/tealus/issues/185)）。

**JSON** を返します（バイナリではありません）。

| フィールド | 内容 |
|---|---|
| `media_count` | そのメッセージの添付の数。複数あるときは `?index=0,1,…` で 1 枚ずつ取る（範囲外は 400） |
| `media` | 全添付のメタ情報（`index` / `mime_type` / `file_name` / `file_size`、本体なし） |
| `index` / `mime_type` / `file_name` / `file_size` | 今回取った 1 件 |
| `data_base64` | 中身（base64）。★ **10MB を超えるときは省き**、`error` に理由が入る |
| `transcription` | 音声メッセージのときだけ。`{ raw_text, formatted_text, status }`（最新版） |

**curl サンプル**:

```bash
curl "http://localhost:3000/api/bot/messages/$MSG_ID/media?index=0" \
  -H "Authorization: Bearer $TOKEN" \
  | jq -r '.data_base64' | base64 -d > output.jpg
```

!!! warning "以前の記述の誤り"
    以前このページは「画像は binary stream で返る」「音声は `?raw=1` で生 wav」と書いていましたが、**どちらも誤り**でした。返事は常に JSON で、`raw` というクエリはありません。

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
    旧形式 `{raw_text, formatted_text, status}` から default 振る舞いが変わります（`raw_text` 省略 + `id` / `version` 追加）。本体側 consumer（`agent-server/src/media/messageAdapter.mts` 等）は `formatted_text || raw_text` fallback chain で機能するため実害なし、むしろ整形空 voice で raw garbage を LLM に渡さなくなる improvement として動きます。

## ヘルスチェック

| メソッド | パス | 説明 | 認証 |
|---|---|---|---|
| GET | `/api/health` | ヘルスチェック | 不要 |
