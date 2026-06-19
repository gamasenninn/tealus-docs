# AIエージェント設定

Tealus の agent-server は「**メッセージ → 適切な処理者へ dispatch**」する hub として機能し、3 種類の処理者に振り分けます（v0.2.2〜「AI 班 dispatcher」化、[#213](https://github.com/gamasenninn/tealus/issues/213)）。

## アーキテクチャ概要

```
[外部 channel]                    [Tealus 本体]
  LINE Official Account ──webhook──┐
                                   ▼
                          ユーザーメッセージ
                                   ↓
                          Router（gpt-5.4-mini）
  ├── 日常タスク（/light）        → Light Agent（default v2、AGENT_LIGHT_BACKEND env で v1/v2 切替、v0.3.0〜）
  ├── cross-room 探索（/light2）  → Light Agent v2（codex SDK、明示指定）
  ├── 複雑タスク（/deep）         → Deep Agent（DEEP_AGENT_PROVIDER で Claude Code CLI / Codex CLI 切替、v0.3.0〜）
  ├── @cc-{project}              → Claude Code session（cc-tealus、v0.2.2〜）
  └── @{alias}（cc-aliases.json）→ Claude Code session
                                   ↓
                          Bot API → Tealus チャット
```

| エージェント | 実装基盤 | 用途 |
|---|---|---|
| Router | gpt-5.4-mini | メッセージの意図分類・複雑度判定 |
| Light Agent v1 | OpenAI Agents SDK（gpt-5.4-mini） | 日常Q&A・翻訳・簡単なタスク（`AGENT_LIGHT_BACKEND=v1` で選択） |
| Light Agent v2（**default**、v0.3.0〜） | codex SDK | cross-room 探索・多角的な tool orchestration（`/light2` prefix、`LIGHTV2_AUTH=subscription` でサブスク認証） |
| Deep Agent — Claude Code CLI（default） | Claude Code CLI（MAX プラン or API key） | コード生成・複雑な分析・長いワークフロー（既存運用） |
| Deep Agent — Codex CLI（v0.3.0〜） | Codex CLI（ChatGPT サブスクリプション or API key） | Anthropic 環境外でも Deep mode 同等運用（`DEEP_AGENT_PROVIDER=codex`、[#276](https://github.com/gamasenninn/tealus/issues/276)） |
| Claude Code session（cc-tealus） | 採用者の手元 Claude Code | `@cc-{project}` / `@{alias}` mention でリアルタイム連携（[#213](https://github.com/gamasenninn/tealus/issues/213)、詳細は [Agent ガイド](../dev/agents.md)） |
| **LINE Bridge** (Inbound、v0.3.0〜) | webhook `POST /api/line` | LINE Official Account → Tealus ルーム投影（7 type 対応：text / image / audio / video / file / sticker / location、詳細は [LINE Bridge 連携](../guide/integration/line-bridge.md)） |

## ゲストユーザ role — Phase 1 〜 D/E 運用可能化（v0.3.0〜）

v0.3.0 で **ゲストユーザ role** が追加 ([#282](https://github.com/gamasenninn/tealus/issues/282) Phase 1 MVP)、v0.3.0 後 (2026-06-16) で **Phase D/E 運用可能化** + 招待ガード fail-closed 修正が反映されました。`users.role` は `admin / user / guest` の 3 種類。

### Phase 1 (v0.3.0) — 3 層構造の基礎

| Layer | 実装 |
|---|---|
| Layer 1 schema | migration 022 で `users.role` CHECK constraint 拡張 |
| Layer 2 logic | `server/src/utils/permissions.js` の `isAdmin/isUser/isGuest` + `canCreateRoom/canInviteToRoom/canSearchUsers` ability check |
| Layer 3 route guard | POST /api/rooms / GET /api/users 等に guest guard、guest → 403 |

### Phase D/E（v0.3.0 後、2026-06-16）— 運用可能化

| 改善 | 詳細 |
|---|---|
| **ゲストにホーム surface を見せない fail-closed**（`ad41620`） | お知らせ / ポータルをゲストには表示しない、`is_published` 等の意図せぬ露出を構造的に防止 |
| **招待ガードの穴を fail-closed 修正**（`4c54dfe`） | ゲスト招待時の権限チェック gap を fail-closed で塞ぐ |
| **`%委譲%` 構文 + 委譲権限 gate**（[#295](https://github.com/gamasenninn/tealus/issues/295) 連動、`592491f`） | bot membership 確認 endpoint で **委譲先メンバーシップを bot 単位で検証**、未参加 room への委譲は拒否（cross-room delegation の権限境界） |

ゲストユーザは **AI = 同僚** 思想の外部 user 拡張、multi-agent dock 構想（[#275](https://github.com/gamasenninn/tealus/issues/275)）の構造基盤。詳細は [Agent ガイド > `%委譲%` syntax](../dev/agents.md#delegation-syntax) も参照。

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

## TTS 読み上げ（音声合成）

AIエージェントの回答を音声で読み上げる機能です。Aivis Cloud API を使用します。

### Aivis Cloud API

音声合成サービスとして [Aivis Cloud API](https://aivis-project.com/) を利用しています。

| プラン | 料金 |
|---|---|
| 従量課金 | ¥440 / 1万文字 |
| 月額定額 | ¥1,980 / 無制限 |

`agent-server/.env` に `AIVIS_API_KEY` を設定してください。

### ルームごとの音声モデル設定

ダッシュボードのルーム設定（基本設定タブ）から、ルームごとに読み上げの音声モデルを選択できます。設定は `room_settings.json` の `tts_model_uuid` フィールドに保存されます。

選択可能なモデルは10種類（凛音エル、まお、にせ、まい、阿井田 茂、fumifumi、morioki、ろてじん、花音、コハク）です。詳細は [TTS 読み上げ](../guide/advanced/tts.md) を参照してください。

## コスト目安

| エージェント | 月額目安 |
|---|---|
| Router | $3〜$13 |
| Light Agent | $80〜$280 |
| Deep Agent | $100（MAXプラン固定） |
| **合計** | **$183〜$393** |

!!! warning
    Deep AgentのコストはMAXプランの固定料金が前提です。API課金に切り替える場合はコストが大幅に変動する可能性があります。
