# Agent ガイド（dispatcher 構造と各 agent の選び分け）

Tealus の agent-server は「**メッセージ → 適切な処理者へ dispatch**」する hub として機能します。本ページは dispatcher 構造の概観と、内蔵 Light / Deep agent の選び分けを扱う実用ガイドです。AI 統合の全体像と Router 判定ロジックは [AI アーキテクチャ](ai-architecture.md) を参照してください。

## Agent dispatcher 構造

agent-server は v0.2.2 で「**AI 班 dispatcher**」化されました（[#213](https://github.com/gamasenninn/tealus/issues/213)）。受信メッセージを以下 3 種類の処理者に dispatch します。

### dispatch 先（3 種類）

| dispatch 先 | 実装基盤 | 役割 | 起動 |
|---|---|---|---|
| **Light agent** | OpenAI Agents SDK（gpt-5.4-mini） | 短い応答、即座の Q&A | プロセス常駐 |
| **Deep agent** | Claude Code CLI | コード生成、複雑タスク、リファクタリング | コマンド毎に CLI を spawn |
| **Claude Code session**（cc-tealus、v0.2.2〜） | 採用者の手元 Claude Code | リアルタイム連携、context 連続性、user 自身の作業空間 | file beacon → Monitor で sub-second wake |

### 進化軸

| バージョン | dispatcher の到達点 |
|---|---|
| v0.1.0 | Light / Deep の 2 層（Router + 2 agents） |
| v0.2.0 | Light agent も Tealus MCP 統合（[#199](https://github.com/gamasenninn/tealus/issues/199)） |
| v0.2.2 | Claude Code session も dispatch 先に追加（cc-tealus Phase A、[#213](https://github.com/gamasenninn/tealus/issues/213)） |
| 将来 | multi-session lock（[#214](https://github.com/gamasenninn/tealus/issues/214) Phase B）など |

cc-tealus は **終点ではなく現時点の到達点**です。「人と AI が同じテーブル」の AI 側を、内蔵 agent だけでなく**採用者の手元で動く Claude Code session** にも開いた、という位置づけです。

## 比較表（Light / Deep）

| 比較軸 | Light Agent | Deep Agent |
|---|---|---|
| 実装基盤 | OpenAI Agents SDK | Claude Code CLI |
| モデル | gpt-5.4-mini | Claude（MAX プラン or API 課金） |
| 起動形態 | プロセス常駐 | コマンド毎に CLI を spawn |
| コンテキスト | 直近 20 件（`LIGHT_CONTEXT_MESSAGES`） | `--resume tealus-room-{roomId}` で継続セッション |
| 月額目安 | $80〜$280 | $100（MAX 固定） / API は変動 |
| 得意領域 | 日常会話、軽量 Q&A、定型タスク、自然言語検索 | コード生成・分析、長いワークフロー、リファクタリング |
| ローカルツール | `write_memory` / `read_memory` / `get_current_time` / `list_workspace_files` / `generate_image` / `code_interpreter` | Claude Code 標準ツール（ファイル編集・bash 実行など） |
| Tealus MCP | 共通 11 ツール（v0.2.0〜、[#199](https://github.com/gamasenninn/tealus/issues/199)） | 共通 11 ツール |
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

利用可能な 11 ツール（[MCP Server](mcp.md) と同一）:

| ツール名 | 用途 |
|---|---|
| `send_message` | テキスト送信 |
| `send_image` | 画像送信（base64） |
| `get_messages` | メッセージ履歴取得 |
| `get_message_media` | メディア取得（画像視認 / 音声文字起こし） |
| `search_messages` | キーワード/タグ/期間/発言者で全文検索 |
| `mark_tag_done` | タグの完了状態（is_done）切替 |
| `create_room` | グループルームを作成（v0.2.0、bot は admin として自動追加） |
| `delete_room` | グループルームをアーカイブ（v0.2.0、creator + solo member のみ） |
| `list_rooms` | 参加中ルーム一覧 |
| `join_room` | ルーム参加 |
| `mark_read` | メッセージ既読化 |

!!! info "外部クライアントとの一貫性"
    Tealus MCP は独立 repo（`gamasenninn/tealus-mcp`）で配布されており、Claude Code など外部 MCP クライアントからも `npx -y github:gamasenninn/tealus-mcp` で利用できます。Tealus 内蔵の Light/Deep agent と同じツールセットなので、開発体験と本番挙動が乖離しません。

## Claude Code session（cc-tealus）

v0.2.2 で agent-server が **採用者の手元 Claude Code session** にも dispatch できるようになりました（[#213](https://github.com/gamasenninn/tealus/issues/213) Phase A）。Tealus 上の `@cc-{project}` mention を検知して file beacon に追記、Claude Code 側が Monitor でファイル変化を検知して sub-second で wake up する仕組みです。

### Phase A（MVP）の動作原理

```
[Tealus] @cc-myproject というメッセージ
     ↓ webhook
[agent-server] @cc- 先頭マッチング → file beacon に append
     ↓
~/.tealus/cc-queue/{project}.jsonl
     ↓ Monitor で監視
[Claude Code session] sub-second で wake up
     ↓ auto_level に従って応答
[Claude Code] suggest reply / 通知 / 自動返信
```

### 採用者向け setup（概要）

1. agent-server を v0.2.2 以降に更新
2. プロジェクトルートに `.claude/cc-tealus.json` を作成（`project_name` / `auto_level` / `queue_path` / `catch_up_policy` を設定）
3. agent-server bot をルームに参加
4. Claude Code session で `/listen-tealus` skill を実行
5. ルームから `@cc-{project}` mention を投稿してテスト

詳細手順は本体 repo の [setup-cc-tealus-bridge.md](https://github.com/gamasenninn/tealus/blob/main/docs/setup-cc-tealus-bridge.md) を参照してください（このページは概念解説、本体側が canonical）。

### auto_level（応答の自動化度合い）

| レベル | 動作 |
|---|---|
| `L1` | 通知のみ |
| **`L2`**（default） | suggest reply（user が承認 / 編集して送信） |
| `L3` | 自動返信（信頼できる FAQ 用途） |

### 論理識別子方式

`project_name` がディレクトリ構造と **decouple** されます。file beacon は `{project}.jsonl` の形で独立管理されるため、同一マシン内なら project ディレクトリの物理位置に依存しません。

### 自己ループ防止（先頭マッチング、[#215](https://github.com/gamasenninn/tealus/issues/215)）

`@cc-{project}` が**メッセージ行頭にある場合のみ** match します。AI reply の本文中に過去 mention が含まれていても、先頭ではないため自然にスキップされます。`CC_SKIP_SENDER_IDS` env で特定 bot の skip 追加も可能です（defense in depth）。

### Phase B 展望

- **multi-session lock**（[#214](https://github.com/gamasenninn/tealus/issues/214)）: 別 PC で同 project を動かす場合の排他制御。現状は共有ストレージ（NAS / SMB）で `~/.tealus/cc-queue/` を mount する運用が想定されています
- 他 dispatch 先の追加候補は本体 roadmap で議論中

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
- [MCP Server](mcp.md): 11 ツールの詳細仕様と外部クライアントからの利用方法
- [AIエージェント設定](../admin/agent.md): 運用者向けの設定手順
- 関連 issue: [#185](https://github.com/gamasenninn/tealus/issues/185) MCP umbrella、[#194](https://github.com/gamasenninn/tealus/issues/194) search_messages、[#197](https://github.com/gamasenninn/tealus/issues/197) mark_tag_done、[#199](https://github.com/gamasenninn/tealus/issues/199) Light agent MCP 統合、[#200](https://github.com/gamasenninn/tealus/issues/200) create_room、[#207](https://github.com/gamasenninn/tealus/issues/207) delete_room、[#213](https://github.com/gamasenninn/tealus/issues/213) cc-tealus Phase A、[#214](https://github.com/gamasenninn/tealus/issues/214) multi-session lock（Phase B）
