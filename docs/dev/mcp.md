# MCP Server

Tealus の Bot API を MCP（Model Context Protocol）ツールとしてラップし、Claude Code や他の MCP クライアントから Tealus をネイティブツールとして操作できます。

## 概要

- リポジトリ: **独立 repo [`gamasenninn/tealus-mcp`](https://github.com/gamasenninn/tealus-mcp)**（v0.6.0）
- 配布: npm publish ではなく **GitHub から直接 install**（`npx -y github:gamasenninn/tealus-mcp`）
- 通信: stdio (MCP) → HTTP (Bot API) → Tealus Server (port 3000)
- 技術スタック: `@modelcontextprotocol/sdk`、`zod`、`node-fetch`、`form-data`
- Tealus 本体（`gamasenninn/tealus`）には移転 stub のみ残存

## 利用可能なツール（11個）

| ツール名 | 説明 | 主な入力パラメータ |
|---|---|---|
| `send_message` | テキストメッセージ送信 | `room_id` / `content` |
| `send_image` | 画像送信（base64） | `room_id` / `image_base64` / `filename` / `caption?` |
| `get_messages` | メッセージ履歴取得 | `room_id` / `limit?`（最大100） |
| `get_message_media` | メディア取得（画像はAIが直接視認、音声は文字起こし優先） | `message_id` |
| `search_messages` | キーワード/タグ/期間/発言者で全文検索（snippetハイライト付） | `q?` / `room_id?` / `sender_id?` / `type?` / `tag_names?` / `is_done?` / `since?` / `until?` / `limit?`（1-50）/ `offset?` |
| `mark_tag_done` | タグの完了状態（is_done）を更新 | `message_id` / `tag_name` / `is_done` |
| `create_room` | グループルームを作成（v0.2.0、bot は admin として自動追加） | `name` / `member_ids?` / `type?`（現状 `'group'` のみ） |
| `delete_room` | グループルームをアーカイブ（v0.2.0、creator + solo member のみ、破壊的・CASCADE 削除） | `room_id` |
| `list_rooms` | 参加中ルーム一覧 | （なし） |
| `join_room` | ルーム参加 | `room_id` |
| `mark_read` | メッセージ既読化 | `message_ids[]` |

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
