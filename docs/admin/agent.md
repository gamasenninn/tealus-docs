# AIエージェント設定

Tealus には3層構造のAIエージェントシステムが統合されています。

## アーキテクチャ概要

```
ユーザーメッセージ
    ↓
  Router（GPT-4o-mini）
  ├── 日常タスク → Light Agent（GPT-4o-mini）
  └── 複雑タスク → Deep Agent（Claude Code）
    ↓
  Bot API → Tealus チャット
```

| エージェント | モデル | 用途 |
|---|---|---|
| Router | GPT-4o-mini | メッセージの意図分類・複雑度判定 |
| Light Agent | GPT-4o-mini | 日常Q&A・翻訳・簡単なタスク |
| Deep Agent | Claude Code（MAXプラン） | コード生成・複雑な分析・長いワークフロー |

## Agent Server

Agent Server は Port 4000 で独立したExpressプロセスとして動作します。Tealus Server（Port 3000）からWebhookでメッセージを受信します。

## 応答ルール

| ルーム種別 | デフォルト動作 |
|---|---|
| ダイレクトメッセージ | すべてのメッセージに応答 |
| グループルーム | `@メンション` 時のみ応答 |

### ルーム単位の設定

`response_mode` で応答動作をルームごとに変更できます。

| モード | 動作 |
|---|---|
| `auto` | DM=全応答、グループ=メンションのみ（デフォルト） |
| `mention` | 常にメンション時のみ応答 |
| `all` | 常にすべてのメッセージに応答 |
| `off` | 応答しない |

設定ファイル: `agent-workspaces/{agentId}/{roomId}/room_settings.json`

## グローバル設定

ダッシュボードまたは設定ファイルの直接編集で管理します。

### settings.json

`agent-server/config/settings.json` でグローバル設定を管理します。

- ツールの有効/無効
- 最大ターン数（`maxTurns`）
- システムプロンプト（`system_prompt`）

### MCP設定

`agent-server/mcp_config.json` でMCP接続を設定します。

## コンテキスト管理

各エージェントのコンテキストは `agent-workspaces/{agent_id}/{room_id}/` にファイルベースで保存されます。

- `CLAUDE.md` — 行動制限・ルール
- `memory/MEMORY.md` — エージェントのメモリ

## 無限ループ防止

Webhook受信時に `sender.is_bot` を確認し、Botからのメッセージは無視します。これによりBot同士の無限応答ループを防止します。

## コスト目安

| エージェント | 月額目安 |
|---|---|
| Router | $3〜$13 |
| Light Agent | $80〜$280 |
| Deep Agent | $100（MAXプラン固定） |
| **合計** | **$183〜$393** |

!!! warning
    Deep AgentのコストはMAXプランの固定料金が前提です。API課金に切り替える場合はコストが大幅に変動する可能性があります。
