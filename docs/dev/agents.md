# Agent ガイド（Light / Deep の選び分け）

Tealus は 2 種類の AI Agent を併用する 3 層構造（Router + Light + Deep）を採用しています。本ページは **どちらの Agent をどのケースで選ぶか** に焦点を当てた実用ガイドです。AI 統合の全体像と Router 判定ロジックは [AI アーキテクチャ](ai-architecture.md) を参照してください。

## 比較表

| 比較軸 | Light Agent | Deep Agent |
|---|---|---|
| 実装基盤 | OpenAI Agents SDK | Claude Code CLI |
| モデル | gpt-5.4-mini | Claude（MAX プラン or API 課金） |
| 起動形態 | プロセス常駐 | コマンド毎に CLI を spawn |
| コンテキスト | 直近 20 件（`LIGHT_CONTEXT_MESSAGES`） | `--resume tealus-room-{roomId}` で継続セッション |
| 月額目安 | $80〜$280 | $100（MAX 固定） / API は変動 |
| 得意領域 | 日常会話、軽量 Q&A、定型タスク、自然言語検索 | コード生成・分析、長いワークフロー、リファクタリング |
| ローカルツール | `write_memory` / `read_memory` / `get_current_time` / `list_workspace_files` / `generate_image` / `code_interpreter` | Claude Code 標準ツール（ファイル編集・bash 実行など） |
| Tealus MCP | 共通 9 ツール（v0.1.x、[#199](https://github.com/gamasenninn/tealus/issues/199)） | 共通 9 ツール |
| タイムアウト | 短め（リアルタイム前提） | 5 分（`DEEP_TIMEOUT`） |

## 選択基準

Router が以下の優先順位で振り分けます。詳細は [AI アーキテクチャ](ai-architecture.md) の「Router — 2段階振り分け」セクションを参照してください。

| 優先度 | 条件 | 振り分け先 |
|---|---|---|
| 1 | `/light` / `/deep` コマンド | 明示指定通り |
| 2 | 挨拶 / 短文（こんにちは、ありがとう等） | Router が直接応答（Agent を呼ばない） |
| 3 | Deep キーワード（コード / リファクタ / デバッグ / 実装 / PR 等） | Deep Agent |
| 4 | 上記すべて該当しない | LLM（gpt-5.4-mini）が `light` / `deep` を判定 |

### 使い分けの実用指針

| 用途 | 推奨 Agent | 理由 |
|---|---|---|
| チャット履歴の検索・要約 | **Light** | レスポンスが速く、コンテキスト 20 件で十分 |
| TODO 管理（完了マーク・一覧） | **Light** | MCP ツール（`mark_tag_done` / `search_messages`）の組み合わせのみで完結 |
| 画像の内容認識・OCR 的依頼 | **Light** | `get_message_media` で画像視認、コード生成は不要 |
| ファイル編集・コードレビュー | **Deep** | Claude Code の編集ツール群が必須 |
| 長いリファクタリング・複数ファイル変更 | **Deep** | 5 分のタイムアウトと `--resume` での継続セッション |
| Issue ↔ コード ↔ 履歴の横断調査 | **Deep** | Tealus MCP + ローカルファイル + Web 検索を組み合わせやすい |

## Tealus MCP 統合（Light/Deep 共通）

両 Agent は **同じ Tealus MCP プロセスを共有**します。`agent-server/src/mcp/roomMcpManager.js` の `getOrCreateSharedGlobal()` で programmatic に注入され、環境変数 `TEALUS_USER_ID` / `TEALUS_PASSWORD` の有無で自動切替されます。

利用可能な 9 ツール（[MCP Server](mcp.md) と同一）:

| ツール名 | 用途 |
|---|---|
| `send_message` | テキスト送信 |
| `send_image` | 画像送信（base64） |
| `get_messages` | メッセージ履歴取得 |
| `get_message_media` | メディア取得（画像視認 / 音声文字起こし） |
| `search_messages` | キーワード/タグ/期間/発言者で全文検索 |
| `mark_tag_done` | タグの完了状態（is_done）切替 |
| `list_rooms` | 参加中ルーム一覧 |
| `join_room` | ルーム参加 |
| `mark_read` | メッセージ既読化 |

!!! info "外部クライアントとの一貫性"
    Tealus MCP は独立 repo（`gamasenninn/tealus-mcp`）で配布されており、Claude Code など外部 MCP クライアントからも `npx -y github:gamasenninn/tealus-mcp` で利用できます。Tealus 内蔵の Light/Deep agent と同じツールセットなので、開発体験と本番挙動が乖離しません。

## システムプロンプトのフロー

両 Agent は共通のシステムプロンプト合成フローを使います。優先度の高い層が低い層を上書きします。

```
default_system_prompt.md         (リポジトリ管理、Tealus MCP 利用ガイダンス内蔵)
        ↓ 上書き（任意）
system_prompt.md                 (.gitignored、user カスタム枠)
        ↓ ルーム個別の上書き（任意）
{workspace}/light_prompt.md      (Light Agent 用)
{workspace}/deep_prompt.md       (Deep Agent 用)
        ↓ メモリ追加
{workspace}/memory/MEMORY.md     (Agent の記憶、長期保存)
```

- **`default_system_prompt.md`**: リポジトリにコミットされる共通プロンプト。Tealus MCP の使い方ガイダンスを含む
- **`system_prompt.md`**: `.gitignore` 対象。運用者ごとに自由カスタマイズする枠
- **ルーム別プロンプト**: ワークスペース配下に置くと、そのルーム内では追加コンテキストとして合成される
- **メモリ**: Agent が `write_memory` ツールで自分宛に書き込む長期記憶

ルーム単位でエージェントの性格・口調を切り替える運用が可能です（声と合わせた管理は [AI アーキテクチャ](ai-architecture.md) の「声は人格の一部」セクションを参照）。

## 関連

- [AI アーキテクチャ](ai-architecture.md): 全体構造、Router の判定ロジック、Webhook Dispatcher、コスト設計
- [MCP Server](mcp.md): 9 ツールの詳細仕様と外部クライアントからの利用方法
- [AIエージェント設定](../admin/agent.md): 運用者向けの設定手順
- 関連 issue: [#185](https://github.com/gamasenninn/tealus/issues/185) MCP umbrella、[#194](https://github.com/gamasenninn/tealus/issues/194) search_messages、[#197](https://github.com/gamasenninn/tealus/issues/197) mark_tag_done、[#199](https://github.com/gamasenninn/tealus/issues/199) Light agent MCP 統合
