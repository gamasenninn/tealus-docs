# MCP Server

Tealus の Bot API を MCP（Model Context Protocol）ツールとしてラップし、Claude Code や他の MCP クライアントから Tealus をネイティブツールとして操作できます。

## 概要

- ディレクトリ: `tealus/mcp-server/`（Agent Server とは独立）
- 通信: stdio (MCP) → HTTP (Bot API) → Tealus Server (port 3000)
- 技術スタック: `@modelcontextprotocol/sdk`、`zod`、`node-fetch`、`form-data`

## 利用可能なツール

| ツール名 | 説明 |
|---|---|
| `send_message` | テキストメッセージ送信 |
| `send_image` | 画像送信（base64入力 → Buffer変換） |
| `get_messages` | メッセージ取得 |
| `list_rooms` | ルーム一覧取得 |
| `join_room` | ルームに参加 |
| `mark_read` | 既読マーク |

## セットアップ

### MCP設定ファイル

Claude Code や他のMCPクライアントの設定ファイル（`mcp_config.json`）に以下を追加します。

```json
{
  "mcpServers": {
    "tealus": {
      "command": "node",
      "args": ["/path/to/tealus/mcp-server/src/index.js"],
      "env": {
        "TEALUS_API_URL": "http://localhost:3000",
        "TEALUS_BOT_ID": "<your_bot_id>",
        "TEALUS_BOT_PASS": "<your_bot_password>"
      }
    }
  }
}
```

!!! warning
    `<your_bot_id>` と `<your_bot_password>` は実際のBotユーザーの認証情報に置き換えてください。`args` のパスも環境に合わせて変更してください。

### 環境変数

| 変数 | 説明 |
|---|---|
| `TEALUS_API_URL` | Tealus Server の URL |
| `TEALUS_BOT_ID` | Bot ユーザーのユーザーID |
| `TEALUS_BOT_PASS` | Bot ユーザーのパスワード |

## 活用例

- Claude Code からチャットにメッセージを投稿
- AIエージェントがチャット履歴を読み取って分析
- 外部AIツールからTealusを操作
