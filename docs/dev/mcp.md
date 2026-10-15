# MCP Server

Tealus の Bot API を MCP（Model Context Protocol）ツールとしてラップし、Claude Code や他の MCP クライアントから Tealus をネイティブツールとして操作できます。

## 概要

- リポジトリ: **独立 repo [`gamasenninn/tealus-mcp`](https://github.com/gamasenninn/tealus-mcp)**（最新 v0.16.0）
- 配布: npm publish ではなく **GitHub から直接 install**（`npx -y github:gamasenninn/tealus-mcp`）
- 通信: stdio (MCP) → HTTP (Bot API) → Tealus Server (port 3000)
- 技術スタック: `@modelcontextprotocol/sdk`、`zod`、`node-fetch`、`form-data`
- Tealus 本体（`gamasenninn/tealus`）の `mcp-server/` は移転を知らせる README だけの名残です（実装は入っていません）

## 利用可能なツール（v0.15.0 以降で 19 個）

!!! warning "見えるツール数は pin しているバージョンで変わります"
    `.mcp.json` で **タグ pin**（`github:gamasenninn/tealus-mcp#v0.14.6` のような形）にしている場合、そのタグ時点のツールセットしか見えません。`send_form` は **v0.14.8 以降**、`edit_message` は **v0.15.0 以降**です。v0.16.0 からは、投稿の決まり（mention の書き方・同報への返し方）を**案内文（instructions）**として接続時に渡します。

!!! danger "★ 本体の AI エージェントが使う版は、最新ではありません"
    本体の agent-server（Light v2 / Deep）は tealus-mcp を **`#v0.14.7` に固定**して起動しています（`agent-server/src/agents/lightV2.mts` / `deep.mts`）。そのため**本体の AI エージェントからは `send_form` と `edit_message` が見えません**。外部の MCP クライアント（Claude Code 等）で最新を使っている場合と、使える道具が違う点に注意してください（本体側で対応を検討中）。

    また、pin を上げたあと **Claude Code をフル再起動**（プロセスごと）しないと反映されません。in-app reload や `/mcp` 再接続では **stdio 子プロセスが生き残り、`npx` が再解決されない**ためです。ref 無し（`github:gamasenninn/tealus-mcp` = HEAD 追従）の場合はフル再起動だけで反映されます。

| ツール名 | 説明 | 主な入力パラメータ |
|---|---|---|
| `send_message` | テキストメッセージ送信 | `room_id` / `content` |
| `send_image` | 画像送信（base64） | `room_id` / `image_base64` / `filename` / `caption?` |
| `send_text_as_file` | 長文 text を file（.txt / .md 等）として投稿（v0.11.0、Light v2 機能 parity） | `room_id` / `content` / `filename` / `mime_type?` / `caption?` |
| `generate_and_send_image` | `gpt-image-1` で画像生成 → ルームに投稿（v0.11.0、`OPENAI_API_KEY` env 必須。v0.5.0 [#313](https://github.com/gamasenninn/tealus/issues/313) で DALL-E 3 から移行、env `OPENAI_IMAGE_MODEL` で上書き可） | `room_id` / `prompt` / `size?`（`1024x1024` / `1536x1024` / `1024x1536` / `auto`） / `caption?` |
| `send_form` | **回答フォームを投稿**（v0.14.8、本体 v0.6.0 [#336](https://github.com/gamasenninn/tealus/issues/336)）。radio（単一選択・補足 text 可）/ text（自由記述）を出し、回答者は 1 ボタンで返信。`reply_mention` を付けると回答が `@cc-<project>` を起動する | `room_id` / `title` / `fields[]`（`id` / `type`（`radio` \| `text`） / `label` / `options?` / `required?` / `multiline?`） / `intro?` / `reply_mention?` / `submit_label?` |
| `edit_message` | **既存メッセージの本文を直す**（v0.15.0、[#394](https://github.com/gamasenninn/tealus/issues/394)）。既定は前後の文字列を指定する部分置換で、置換した箇所数を返す。`content` で全文差し替え（排他）。`expected_count` で件数を検算、`dry_run` で書き込まず確認。編集できるかはルームの「メッセージ編集」設定で決まり、不可なら 403。旧版は編集履歴に残る | `message_id` / `before?` / `after?` / `content?` / `expected_count?` / `dry_run?` |
| `get_messages` | メッセージ履歴取得 | `room_id` / `limit?`（最大100） / `include_transcription?`（v0.7.0、default true） / `include_raw?`（v0.7.0、default false） |
| `get_message_media` | メディア取得（画像はAIが直接視認、音声は文字起こし優先、v0.14.1 で image 判定を mime_type ベースに緩和）。**複数添付は `index` で 1 枚ずつ取得**（v0.14.5、[#316](https://github.com/gamasenninn/tealus/issues/316)。返り値の `media_count` を見て `index=0,1,…` で全枚取得） | `message_id` / `index?`（0始まり） |
| `get_message_edit_history` | メッセージ編集履歴を取得（text edits + voice / video transcription versions を統合返却、v0.14.0、organon daily cycle の edit history 観察用、[#281](https://github.com/gamasenninn/tealus/issues/281) follow-up） | `message_id` |
| `read_document` | 添付 PDF / DOCX / XLSX / Markdown / Plain Text / CSV を text 化（v0.8.0〜、v0.9.0 で Gemini Vision fallback、v0.11.1 で parseError robustness、v0.13.1 で text/* mime 対応）。**複数添付は `index` で 1 件ずつ取得**（v0.14.6、[#317](https://github.com/gamasenninn/tealus/issues/317)） | `message_id` / `index?`（0始まり） |
| `transcribe_media` | 動画/音声メッセージを文字起こし（v0.13.0、server-side で ffmpeg audio 抽出 → Whisper → AI 整形。`get_message_media` の 10MB 上限を回避） | `message_id` / `force_retranscribe?` |
| `search_messages` | キーワード/タグ/期間/発言者で全文検索（snippetハイライト付）。**`has_reaction` でリアクション有無を SQL 側で絞れる**（v0.14.7、[#325](https://github.com/gamasenninn/tealus/issues/325)。「✅ が付いていないもの一覧」等） | `q?` / `room_id?` / `sender_id?` / `type?` / `tag_names?` / `is_done?` / `has_reaction?` / `since?` / `until?` / `limit?`（1-50）/ `offset?` |
| `list_tags` | bot メンバー全 room の tag 一覧を usage 順で返す discovery primitive（v0.10.0） | `limit?`（default 30、max 100） |
| `mark_tag_done` | タグの完了状態（is_done）を更新 | `message_id` / `tag_name` / `is_done` |
| `create_room` | グループルームを作成（v0.2.0、bot は admin として自動追加） | `name` / `member_ids?` / `type?`（現状 `'group'` のみ） |
| `delete_room` | グループルームをアーカイブ（v0.2.0、creator + solo member のみ、破壊的・CASCADE 削除） | `room_id` |
| `list_rooms` | 参加中ルーム一覧 | （なし） |
| `join_room` | ルーム参加 | `room_id` |
| `mark_read` | メッセージ既読化 | `message_ids[]` |

!!! info "get_messages の transcription verbosity 制御（v0.7.0〜）"
    voice メッセージの文字起こしを `include_transcription` / `include_raw` で 3 段階に出し分けできます。

    | 組合せ | `transcription` field |
    |---|---|
    | `include_transcription=false` | `{id, status, version}`（id-only、軽量モード） |
    | **default**（`include_raw=false`） | `{id, status, version, formatted_text}` |
    | `include_raw=true` | `{id, status, version, formatted_text, raw_text}` |

    AI が要約したいときは default、生 wav の文字起こし全文が必要なときだけ `include_raw=true` という使い分けが推奨です。

!!! info "search_messages の使い方"
    `q` / `room_id` / `sender_id` / `since` / `tag_names` / `type` のうち少なくとも1つを指定する必要があります。日本語2文字のクエリは index が効きにくいため**3文字以上推奨**。snippet を索引として使い、詳細が必要なら `get_messages` で再取得する設計です。

## セットアップ

### MCP 設定ファイル

Claude Code や他の MCP クライアントの設定ファイル（`mcp_config.json` など）に以下を追加します。

```json
{
  "mcpServers": {
    "tealus": {
      "command": "npx",
      "args": ["-y", "github:gamasenninn/tealus-mcp"],
      "env": {
        "TEALUS_API_URL": "http://localhost:3000",
        "TEALUS_USER_ID": "<your_bot_id>",
        "TEALUS_PASSWORD": "<your_bot_password>"
      }
    }
  }
}
```

!!! warning
    `<your_bot_id>` と `<your_bot_password>` は実際の Bot ユーザーの認証情報に置き換えてください。`TEALUS_API_URL` は本番運用時はサーバーの実 URL（例: `https://your-domain.example`）を指定してください。

### 環境変数

| 変数 | 必須 | 説明 |
|---|---|---|
| `TEALUS_API_URL` | × | Tealus Server の URL（デフォルト: `http://localhost:3000`） |
| `TEALUS_USER_ID` | ○ | Bot ユーザーのユーザーID |
| `TEALUS_PASSWORD` | ○ | Bot ユーザーのパスワード |

!!! note "旧環境変数名との互換"
    旧名 `TEALUS_BOT_ID` / `TEALUS_BOT_PASS` も内部で fallback されますが、新規セットアップでは `TEALUS_USER_ID` / `TEALUS_PASSWORD` の使用を推奨します。

!!! warning "`TEALUS_` prefix なしの env 名は silent fail（**全バージョン共通**、[tealus #267](https://github.com/gamasenninn/tealus/issues/267)）"
    tealus-mcp は **`TEALUS_` prefix 付きの env のみ** を読み込みます（multi-MCP 環境での衝突回避設計）。`API_URL` / `BOT_ID` / `BOT_PASS` のような prefix なし短縮名は無視され、`TEALUS_API_URL` は default の `http://localhost:3000` に fallback します。

    | ❌ 誤り | ✅ 正しい |
    |---|---|
    | `API_URL=...` | `TEALUS_API_URL=...` |
    | `BOT_ID=...` | `TEALUS_USER_ID=...`（旧 `TEALUS_BOT_ID` も互換） |
    | `BOT_PASS=...` | `TEALUS_PASSWORD=...`（旧 `TEALUS_BOT_PASS` も互換） |

    症状としては、認証が通らず接続エラーになる、または接続先がローカル開発環境にずれます。設定変更後は MCP クライアント（Claude Code 等）の **再起動** が必要です。

## HTTP transport（cross-machine 構成、v0.12.0+） {#http-transport}

tealus-mcp は **stdio**（default）と **HTTP**（opt-in）の 2 transport を提供します（[tealus #264](https://github.com/gamasenninn/tealus/issues/264) Phase 1 alpha）。stdio path は **既存採用者環境を無変更で動かす** ために default 維持、HTTP は **cross-machine 構成**（Tealus 本体と MCP クライアントが別マシン）で必要な選択肢です。

### 構成図

```
[Claude Code (マシン A)]              [Tealus サーバ (マシン B)]
  ~/.claude.json                          port 3000 (Tealus 本体)
  mcpServers:                             ┌──────────────────┐
    tealus:                               │ /mcp proxy       │
      url: https://<host>/mcp             │   ↓              │
      headers:                            │ port 3200        │
        Authorization: Bearer <JWT> ────► │ tealus-mcp HTTP  │
                                          └──────────────────┘
```

### v0.12.x の進化

| version | 内容 |
|---|---|
| **v0.12.0**（5/8） | `StreamableHTTPServerTransport` 導入、初版 |
| **v0.12.1**（5/10） | dotenv で `.env` 読み込み（host mode 友好度向上） |
| **v0.12.2**（5/10） | `/mcp/health` endpoint 追加（proxy 経由 reachability check） |
| **v0.12.3**（5/10） | **stateful session 管理**（proper MCP client = Claude Code 対応の構造 fix、`tools/list` failure を解消） |

### 認証と JWT_SECRET 共有

HTTP transport は **JWT auth** を必須とし、`JWT_SECRET` を **Tealus 本体 server / agent-server / tealus-mcp の 3 process で完全同値** にする運用です。proxy は pass-through、検証は tealus-mcp 側で fail-fast 401。

長寿命 JWT（例: 30 日 expiry）を発行して `~/.claude.json` の `headers.Authorization` に設定します。

### Phase 1 alpha の scope と限界

| 項目 | 状態 |
|---|---|
| **HTTP tools 呼び出し**（Outbound） | ✅ Phase 1 alpha で動作（Claude Code から `tealus-http · ✔ connected · 15 tools` を verify 済） |
| **mention 通知 wake-up**（Inbound） | stdio + file beacon 経路に依存。HTTP では **未サポート**（Phase 2 で SSE event broker を予定） |
| **stdio との関係** | stdio は v0.12.0+ でも維持、既存採用者環境は無変更。HTTP は `--transport=http` flag + JWT_SECRET 設定で opt-in |

cross-machine 構成で **HTTP + cc-tealus stdio bridge 併用** が当面の現実解です。

### 採用者向け詳細手順

step-by-step は本体 repo の **[`setup-cc-tealus-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-cc-tealus-bridge.md) のステップ 5A** に集約しています（HTTP server 起動 / `/mcp` proxy 動作確認 / JWT 発行 / `~/.claude.json` url-based entry 追加 / 動作確認）。

採用者向けの選び分け概念紹介は [Claude Code 連携](../guide/integration/claude-code.md) を参照してください。

!!! warning "HTTPS / TLS 終端"
    tealus host を public expose する場合は **HTTPS / reverse proxy（nginx 等）で TLS 終端必須**。`<JWT>` は適切な expiry で運用、漏洩時は `JWT_SECRET` rotate で全 token 失効。

## 活用例

### AIエージェントからチャット投稿

Claude Code やその他の MCP クライアントから、業務メモやチームルームに直接メッセージを投稿できます。

### AIによる画像視認

`get_message_media` で画像メッセージを取得すると、MCP の image content として返却され、AI が画像内容を直接認識できます。スクリーンショットの解析やレシート OCR、現場写真の状況判断などに活用できます。

### 全文検索による議論の振り返り

`search_messages` でキーワード・期間・発言者・タグの組合せで議論を検索できます。snippet を索引として使い、必要があれば `get_messages` で詳細を取得する 2 段階アクセスが推奨パターンです。

```
例: 「先週の議論を search_messages で検索して」
    → AIが q/since/until を組み立てて呼び出し
    → snippet で要約 → ユーザーが気になった箇所だけ get_messages で深掘り
```

### TODO 完了マーク

`mark_tag_done` でメッセージのタグの完了状態を切り替えられます。voice メモから自動抽出した TODO を AI が完了化する、といったワークフローに活用できます。

### AI による能動的なルーム編成（v0.2.0）

`create_room` / `delete_room` により、AI が**読む**だけでなく**ルーム編成にも参加**できるようになりました（v0.2.0、[#200](https://github.com/gamasenninn/tealus/issues/200) / [#207](https://github.com/gamasenninn/tealus/issues/207)）。「議題ごとに新しいルームを作って整理して」「役目を終えたルームをアーカイブして」のような依頼を AI が直接実行できます。

`delete_room` には安全制約として **creator のみ + solo member のみ + グループルーム限定** が課されています（破壊的操作のため）。

### Light agent / Deep agent 共有

Tealus の AI エージェント（Light/Deep の 2 層）はいずれもこの同一 MCP を共有します。Claude Code のような外部クライアントと、Tealus 内蔵 AI が**同じツールセット**で動作するため、開発者と本番 AI の体験が乖離しません。

## 関連 issue（tealus 本体）

- [#187 tealus-mcp 独立 repo 化](https://github.com/gamasenninn/tealus/issues/187)
- [#194 search_messages MCP tool](https://github.com/gamasenninn/tealus/issues/194)
- [#197 mark_tag_done MCP tool](https://github.com/gamasenninn/tealus/issues/197)
- [#199 Light agent も Tealus MCP 統合](https://github.com/gamasenninn/tealus/issues/199)
- [#200 create_room MCP tool](https://github.com/gamasenninn/tealus/issues/200)
- [#207 delete_room MCP tool](https://github.com/gamasenninn/tealus/issues/207)
- [#232 read_document MCP tool（PDF/DOCX/XLSX 解析、v0.8.0）](https://github.com/gamasenninn/tealus/issues/232)
- [#233 read_document に Gemini Vision fallback（scan PDF 対応、v0.9.0）](https://github.com/gamasenninn/tealus/issues/233)
- [#254 list_tags MCP tool（v0.10.0）](https://github.com/gamasenninn/tealus/issues/254)
- [#260 send_text_as_file / generate_and_send_image（Light v2 機能 parity、v0.11.0）](https://github.com/gamasenninn/tealus/issues/260)
- [#262 E2E harness で v0.11.1 parseError → vision fallback 発覚](https://github.com/gamasenninn/tealus/issues/262)
- [#271 transcribe_media MCP tool（動画/音声 文字起こし、v0.13.0）](https://github.com/gamasenninn/tealus/issues/271)
- [#281 read_document text/* mime 対応（v0.13.1）](https://github.com/gamasenninn/tealus/issues/281)
- [#208 get_message_edit_history MCP tool（v0.14.0、5/24 Day 8 user 提案、organon daily cycle で edit history 観察用）](https://github.com/gamasenninn/tealus/issues/208)
- [#292 get_message_media image 判定を mime_type ベースに緩和（v0.14.1）](https://github.com/gamasenninn/tealus/issues/292)
