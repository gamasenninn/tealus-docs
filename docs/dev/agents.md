# Agent ガイド（dispatcher 構造と各 agent の選び分け）

Tealus の agent-server は「**メッセージ → 適切な処理者へ dispatch**」する hub として機能します。本ページは dispatcher 構造の概観と、内蔵 Light / Deep agent の選び分けを扱う実用ガイドです。AI 統合の全体像と Router 判定ロジックは [AI アーキテクチャ](ai-architecture.md) を参照してください。

## Agent dispatcher 構造

agent-server は v0.2.2 で「**AI 班 dispatcher**」化されました（[#213](https://github.com/gamasenninn/tealus/issues/213)）。受信メッセージを以下 3 種類の処理者に dispatch します。

### dispatch 先（3 種類）

| dispatch 先 | 実装基盤 | 役割 | 起動 |
|---|---|---|---|
| **Light agent** | OpenAI Agents SDK（v1）/ codex SDK（v2、`/light2` prefix） | 短い応答、即座の Q&A、cross-room 探索（v2） | プロセス常駐（v1）/ ephemeral subprocess via SDK（v2） |
| **Deep agent** | Claude Code CLI | コード生成、複雑タスク、リファクタリング | コマンド毎に CLI を spawn |
| **Claude Code session**（cc-tealus、v0.2.2〜） | 採用者の手元 Claude Code | リアルタイム連携、context 連続性、user 自身の作業空間 | file beacon → Monitor で sub-second wake |

### 進化軸

| バージョン | dispatcher の到達点 |
|---|---|
| v0.1.0 | Light / Deep の 2 層（Router + 2 agents） |
| v0.2.0 | Light agent も Tealus MCP 統合（[#199](https://github.com/gamasenninn/tealus/issues/199)） |
| v0.2.2 | Claude Code session も dispatch 先に追加（cc-tealus Phase A、[#213](https://github.com/gamasenninn/tealus/issues/213)） |
| `[Unreleased]`（Phase 4） | **Light v2（codex SDK backed）** を `/light2` prefix で並列追加（[#258](https://github.com/gamasenninn/tealus/issues/258)）、`cc-aliases.json` で `@Claude` 等の alias を設定ファイル化（[#263](https://github.com/gamasenninn/tealus/issues/263)） |
| 将来 | multi-session lock（[#214](https://github.com/gamasenninn/tealus/issues/214) Phase B）など |

cc-tealus は **終点ではなく現時点の到達点**です。「人と AI が同じテーブル」の AI 側を、内蔵 agent だけでなく**採用者の手元で動く Claude Code session** にも開いた、という位置づけです。Light agent は v0.1.0 の OpenAI Agents SDK 単一実装から、v2（codex SDK）併設への並列進化に移行しています（用途別の選び分けは [Light v1 / v2 の選び分け](#light-v1-v2) 参照）。

## 比較表（Light / Deep）

Light は v0.1.0 以来の OpenAI Agents SDK 実装（**Light v1**）に加え、`[Unreleased]`（Phase 4）で codex SDK backed の **Light v2**（`/light2` prefix）が並列追加されました。下表は **Light v1** を代表値として記載しています。Light v1 と v2 の選び分けは [Light v1 / v2 の選び分け](#light-v1-v2) を参照してください。

| 比較軸 | Light Agent | Deep Agent |
|---|---|---|
| 実装基盤 | OpenAI Agents SDK（v1）/ codex SDK（v2、`/light2`） | Claude Code CLI（default）/ Codex CLI（`DEEP_AGENT_PROVIDER=codex`、v0.3.0〜） |
| モデル | gpt-5.4-mini | Claude（MAX プラン or API 課金）/ Codex（ChatGPT サブスクリプション or API 課金） |
| 起動形態 | プロセス常駐（v1）/ ephemeral subprocess via SDK（v2） | コマンド毎に CLI を spawn |
| コンテキスト | 直近 20 件（`LIGHT_CONTEXT_MESSAGES`） | `--resume tealus-room-{roomId}` で継続セッション |
| 月額目安 | v1 $80〜$280 / v2 はサブスク経路あり（後述） | $100（MAX 固定） / API は変動 |
| 得意領域 | 日常会話、軽量 Q&A、定型タスク、自然言語検索、**cross-room 探索（v2 が強い）** | コード生成・分析、長いワークフロー、リファクタリング |
| ローカルツール | `write_memory` / `read_memory` / `get_current_time` / `list_workspace_files` / `generate_image` / `code_interpreter` | Claude Code 標準ツール（ファイル編集・bash 実行など） |
| Tealus MCP | 共通 18 ツール（v0.2.0〜、tealus-mcp v0.14.8 以降で 18、[#199](https://github.com/gamasenninn/tealus/issues/199)） | 共通 18 ツール |
| タイムアウト | 短め（リアルタイム前提） | **8 分**（`DEEP_TIMEOUT`、v0.5.0 [#314](https://github.com/gamasenninn/tealus/issues/314) で延長） |

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

## Light v1 / v2 の選び分け {#light-v1-v2}

`[Unreleased]`（Phase 4）で **Light v2（codex SDK backed）** が `/light2` prefix で並列追加されました（[#258](https://github.com/gamasenninn/tealus/issues/258)）。Light v1（OpenAI Agents SDK 実装）と並走し、用途別に使い分けます。

| 比較軸 | Light v1 | Light v2 |
|---|---|---|
| 実装基盤 | OpenAI Agents SDK | codex SDK（`@openai/codex-sdk` v0.128.0+） |
| 起動形態 | プロセス常駐 | ephemeral subprocess via SDK（CLI spawn は SDK が hide） |
| 認証 | API key（`OPENAI_API_KEY`） | API key OR **ChatGPT サブスクリプション**（`LIGHTV2_AUTH=subscription`） |
| 単純会話 cost | 低（基準） | 約 2x（input token が膨らむ） |
| cross-room 探索 cost | 中（基準） | 約 4x、ただし完結率が明確に高い |
| 得意領域 | 単純会話 / 単 room 要約（cost 効率良） | **cross-room 探索 / 多角的な tool orchestration**（list_rooms → search_messages → get_messages → list_tags の chain で完結率 ↑） |
| 既知の弱点 | task decomposition が浅い、failure recovery が弱い | post-turn の cleanup warn（応答に影響なし） |

### 起動とコマンド prefix

| コマンド | 振り分け先 |
|---|---|
| `/light` | Light（**default backend は v2**、v0.3.0 以降。`AGENT_LIGHT_BACKEND` env で `v1` / `v2` 切替可） |
| `/light2` | Light v2（codex SDK、backend 設定を無視して明示指定） |
| `/deep` | Deep agent（default は Claude Code、`DEEP_AGENT_PROVIDER=codex` で **Codex Deep** に切替可） |
| なし（自動判定） | Router が `light` / `deep` を判定（`light` 時は `AGENT_LIGHT_BACKEND` の default に従う） |

### `AGENT_LIGHT_BACKEND` env — Light backend 設定ファイル化（v0.3.0〜、[#292](https://github.com/gamasenninn/tealus/issues/292)）

v0.3.0 cut 後（6/8-6/9）に **Light backend が config 化** され、`/light` の default backend を env で切り替えられるようになりました。同時に **default が v1 → v2 に flip**（サブスクリプション保有 user の voice 起点、6/9 commit `d0e9645`）。

| 値 | 動作 |
|---|---|
| `v2`（**default**、v0.3.0〜） | `/light` は Light v2（codex SDK）で動作。`/light2` と同等 |
| `v1` | `/light` は Light v1（OpenAI Agents SDK）で動作（v0.2.4 以前の挙動） |

設定例:

```bash
# .env
AGENT_LIGHT_BACKEND=v2   # v0.3.0〜 default
# AGENT_LIGHT_BACKEND=v1 # 旧 default に戻す
```

`/light2` (明示指定 prefix) は **常に Light v2** で動作するため、v1/v2 を併用したい場合に有用です。

!!! info "default flip の背景"
    v0.3.0 後の dogfood で **「サブスクリプション保有なら v2 が当たり前」** という採用者 voice が surface。v2 の cross-room 探索完結率の優位性と組み合わせ、新規 onboarding は v2 default が自然と判断され flip しました。v1 は **cost 効率重視** や **既存 prompt 投資の継続活用** のための opt-in 選択肢として残存します。

### `LIGHTV2_AUTH` env

| 値 | 認証 path | 備考 |
|---|---|---|
| `api_key`（default） | `OPENAI_API_KEY` 経由で API 課金 | v1 と同じ料金体系 |
| `subscription` | ChatGPT サブスクリプション認証 | ChatGPT Plus / Pro 持ちなら API cost 0、ただし生成・画像系は別途 `OPENAI_API_KEY` が必要 |

!!! warning "ChatGPT サブスクリプション path は前提化しすぎない"
    `LIGHTV2_AUTH=subscription` は ChatGPT Plus 等のサブスクリプションを持っている採用者に対して **API cost を 0 にできる選択肢** ですが、サブスクリプションを持たない採用者には適用できません。default は `api_key` のままです。新規 onboarding では「サブスクリプション持ちなら subscription path も選べる」程度の紹介に留めるのが推奨です。

### 推奨運用

| 用途 | tier |
|---|---|
| 単純会話 / 単 room 要約 | **Light v1**（cost 効率、quality 同等） |
| **cross-room / 多角探索** | **Light v2**（完結率明確に勝る） |
| ChatGPT Plus / Pro 持ち | **Light v2**（subscription path で API cost 0） |
| コード生成 / 長いワークフロー | **Deep**（Light v1 / v2 とは別軸） |

!!! info "なぜ並列実装か"
    [#220](https://github.com/gamasenninn/tealus/issues/220) で議論された「task decomposition が浅い」等の Light v1 の構造的弱点に対し、prompt 改善には天井があるという観察から、**codex CLI 内蔵の reasoning + tool orchestration が structurally 強い** ことが verify で実証されました。prompt 投資（v1 改善）と SDK 入替（v2）の trade-off の上で、両者を並列保持し用途で選び分ける形に着地しています。

### Cross-room delegation primitive（v0.3.0〜、[#292](https://github.com/gamasenninn/tealus/issues/292) follow-up） {#cross-room-delegation}

Light v2 が他 room の context を参照しながら応答を組み立てる際の **in-flight tracking + safety net** primitive が v0.3.0 後 (6/13) に追加されました。Light v2 の cross-room 探索完結率を支える内部 mechanism です。

- 自 room への `send_message` と最終 auto-post の重複防止（`fd52b3c`）
- delegation in-flight tracking で **複数 room を跨ぐ tool call の状態管理**
- safety net で in-flight 滞留時の応答保護

採用者からは通常意識不要、cross-room タスクで「応答が重複した」「room A で頼んだのに room B でも post された」等の症状があれば本 primitive の効果を確認してください。

### `%委譲%` syntax — multi-room delegation primitive（v0.3.0 後、[#295](https://github.com/gamasenninn/tealus/issues/295)） {#delegation-syntax}

v0.3.0 後 (2026-06-14〜) で、room 間 AI 委譲を **指示文の語感に依存せず明示構文で安定検出** する `%委譲%` syntax と、専用デリゲーター層が追加されました（[#295](https://github.com/gamasenninn/tealus/issues/295)、Phase 5 narrative core）。

#### 設計の核 — 2 層分離

**第 1 層: 意図の宣言**

| 方法 | 例 | 検出 |
|---|---|---|
| **`%委譲%` 明示構文** | `%委譲%社内DB検索 今月納品になっていないものは何件?` | dispatcher が LLM に渡す前に正規表現で確定パース、**100% 確実 / LLM 非依存** |
| **複数 room fan-out**（`3a66ddc`） | `%委譲%社内DB検索 %委譲%朝礼 ...` | 複数 room へ並列 fan-out → 結果を統合 |
| NL フォールバック | 「社内DB 検索ルームに依頼」等の自然文 | `delegate_to_room(room, task)` tool 経由（明示構文の slow path） |

**第 2 層: 実行（専用デリゲーター）**

cross-room 送信を **すべてデリゲーター経由** にして「考える人」と「他室へ送る人」を分離：

- アシスタント agent は cross-room 投稿を一切しない
- デリゲーターが専用送信者 ID（side channel）から送信 → agent は webhook を誘発する投稿を作らない → **自己ループの種が構造的に消滅**
- [#292](https://github.com/gamasenninn/tealus/issues/292) の in-flight tracking / throttle / 自室送信 skip flag は primary 防御から backstop に格下げ

#### 機能

| 機能 | 説明 |
|---|---|
| **correlation id** | 依頼 ID を封筒に付与、応答 ID を照合して機械的に origin 室へ届ける（LLM の善意に依存しない戻り先 routing） |
| **TTL / サイクルガード** | 封筒に hop 数 + 訪問済み room 集合、深さ超過 / 再訪なら委譲拒否（[#292](https://github.com/gamasenninn/tealus/issues/292) で観測された 1789 件暴走の原因クラス封じ込め） |
| **非同期 + pending 状態** | 「問い合わせ中」ステータス表示（`25c240a`）、対象室 timeout 対応 |
| **DeepCodex 対応** | `route()` ベース backend 選択で Light v2 / Deep Codex 両 path 統一（`8f2bf62`） |
| **権限チェック gate**（[#282](https://github.com/gamasenninn/tealus/issues/282) 連動） | 委譲先メンバーシップを bot membership endpoint で確認、未参加 room への委譲は拒否 |

#### 採用者観点 — room = capability の境界

「全 agent に全 MCP を持たせれば委譲不要」ではない設計判断:

- 社内 DB アクセスは DB 室の Deep Codex (`dbhub-mysql` 所持) にスコープ → **データ主権 / 権限分離** を意図的に保持
- **room = capability の境界、デリゲーター = スコープされた agent を仲介するブローカー**
- 委譲は「権限分離された agent を合成する正しい手段」
- `%委譲%` は「能力境界を越える明示的な依頼」のマーカー

[#282](https://github.com/gamasenninn/tealus/issues/282)（role / permission Phase D/E）と直結、Phase 5 multi-agent dock 構想（[#275](https://github.com/gamasenninn/tealus/issues/275)）の構造的基盤としても機能します。

!!! tip "dogfood 例"
    テスト室で `%委譲%社内DB検索 今月納品になっていないものは何件?` を投稿 → デリゲーターが社内DB検索室の Deep Codex に依頼を送信 → DB Deep Codex が SQL 実行 + 結果生成 → correlation id 経由でテスト室に応答が機械的に戻る、という **「テスト室 →(依頼)→ 社内DB検索室 →(回答)→ テスト室」の往復** が成立。`%委譲%` の前は「で」「に」「依頼」等の語感で判定が割れていた挙動が、構文レベルで安定化されました。

## Deep agent — provider 切替（v0.3.0〜、[#276](https://github.com/gamasenninn/tealus/issues/276)） {#deep-provider}

v0.3.0 (2026-06-01) で Deep agent の **provider 切替** が導入されました。既存の Claude Code CLI に加え、**Codex CLI** で Deep agent を起動できるようになっています。

### 起動方式

| `DEEP_AGENT_PROVIDER` env | 実装 | 認証 |
|---|---|---|
| `claude`（**default**、v0.2.x まで唯一の選択肢） | `claude -p` spawn（既存 Deep agent、`agents/deep.js`） | Claude MAX プラン / API key |
| `codex`（v0.3.0〜、新規） | `codex exec` spawn（新規 `agents/deepCodex.js`、Light v2 `#258` で確立した Codex SDK / subscription auth pattern を CLI mode で再現） | **ChatGPT サブスクリプション**（`~/.codex/auth.json`）/ API key |

### Codex Deep の認証 path — `DEEP_CODEX_AUTH` env

| 値 | 認証 path | 備考 |
|---|---|---|
| `subscription`（**default**） | `~/.codex/auth.json` 経由 ChatGPT subscription | **API cost 0**（Deep mode の 12-50k tokens × call 課金爆発 risk 回避） |
| `api-key` | `OPENAI_API_KEY` 経由で API 課金 | サブスクリプション無しの環境向け |

### Workspace + AGENTS.md

Codex Deep agent は workspace 配下に `.codex_home/` を動的生成し、`auth.json` copy + `config.toml` を room-specific に組み立てます。Codex CLI は project root の `AGENTS.md`（上限 32 KB）を自動 read するため、`CLAUDE.md` の literal copy として `AGENTS.md` を repo root に配置しています。

### 推奨運用

| 用途 | provider |
|---|---|
| Anthropic 環境 / Claude MAX プラン保有 | **claude**（既存運用維持、コード生成 task で安定） |
| OpenAI 環境 / ChatGPT サブスクリプション保有 | **codex**（API cost 0、subscription path の最大活用） |
| dogfood 比較 | 両 provider 並列、room ごとに切り替え |

!!! info "5/16 4 round → 0 round transition (Phase 4 organon paradigm operational realization)"
    Codex Deep agent + organon polyseme inject pipeline ([後述](#system-prompt))の組み合わせで、5/16 dogfood の「4 round LLM 訂正 → 1 round 正解」transition が **両 agent path で再現**（N=2 evidence、5/31 dogfood）。Deep agent の **provider 選択肢拡大が organon-driven query precision を Anthropic 環境外にも拡げる** structural change です。

### Codex 認証エラー検出 helper（v0.3.0 後、6/8 pre-α）

Codex Deep / Light v2 の **認証エラー (`auth.json` 期限切れ / 不正)** を helper が検出し、明示的なエラーメッセージで採用者を guide します（detection only、自動 recovery はしません）。`.env` 設定誤りや `codex login` 期限切れ時の troubleshoot を高速化する design。

## Tealus MCP 統合（Light/Deep 共通）

両 Agent は **同じ Tealus MCP プロセスを共有**します。`agent-server/src/mcp/roomMcpManager.js` の `getOrCreateSharedGlobal()` で programmatic に注入され、環境変数 `TEALUS_USER_ID` / `TEALUS_PASSWORD` の有無で自動切替されます。

利用可能な 18 ツール（[MCP Server](mcp.md) と同一、v0.14.8 以降）:

| ツール名 | 用途 |
|---|---|
| `send_message` | テキスト送信 |
| `send_image` | 画像送信（base64） |
| `send_text_as_file` | 長文 text を file として投稿（v0.11.0） |
| `generate_and_send_image` | `gpt-image-1` で画像生成 → 投稿（v0.11.0、v0.5.0 [#313](https://github.com/gamasenninn/tealus/issues/313) で DALL-E 3 から移行） |
| `send_form` | 回答フォームを投稿（tealus-mcp v0.14.8、本体 v0.6.0 [#336](https://github.com/gamasenninn/tealus/issues/336)） |
| `get_messages` | メッセージ履歴取得 |
| `get_message_media` | メディア取得（画像視認 / 音声文字起こし、v0.14.1 で image 判定を mime_type ベースに緩和） |
| `get_message_edit_history` | メッセージ編集履歴を取得（text edits + voice/video transcription versions、v0.14.0） |
| `read_document` | 添付 PDF / DOCX / XLSX / Markdown / Plain Text / CSV を text 化（v0.8.0〜、v0.9.0 で Gemini Vision fallback、v0.13.1 で text/* mime 対応） |
| `transcribe_media` | 動画/音声メッセージを文字起こし（v0.13.0、`get_message_media` の 10MB 上限を回避） |
| `search_messages` | キーワード/タグ/期間/発言者で全文検索 |
| `list_tags` | bot 全 room の tag 一覧を usage 順で返す（v0.10.0） |
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

### `cc-aliases.json` — alias を設定ファイル化（`[Unreleased]`、[#263](https://github.com/gamasenninn/tealus/issues/263)） {#cc-aliases-json}

`@cc-{project}` 以外に、より自然な mention 名（例: `@Claude`、`@AI`）からも同じ queue へ routing できるようにする alias 機構です。code 変更不要で alias を追加できます。

**設定ファイル**: `agent-server/config/cc-aliases.json`

```json
{
  "aliases": [
    { "mention": "Claude", "project": "tealus" },
    { "mention": "AI", "project": "tealus" }
  ]
}
```

- `mention`: `@` を除いた alias 名（case-insensitive、word boundary 一致）
- `project`: routing 先の cc-queue project（`~/.tealus/cc-queue/{project}.jsonl` に append）
- agent-server 再起動で反映

**優先順位と routing**:

1. `@cc-{project}` 形式 — 既存 path で routing（最優先）
2. `@{alias}` 形式 — `cc-aliases.json` の登録 alias と一致すれば対応 project へ routing
3. それ以外 — Router の通常 dispatch（Light / Deep）

**互換と robustness**:

- file 不在 / parse 失敗時は graceful degrade（alias 0 件、`@cc-*` path は健在）
- invalid entries（空 `mention` / null 等）は skip + 警告 log
- 特殊文字 mention 名（例: `Bot.X+Y`）は regex escape で安全
- 旧 `CLAUDE_DEFAULT_PROJECT` env も legacy fallback として残存

!!! info "採用者 UX の改善"
    `@Claude` のような自然な mention 名は、Tealus を **OSS として配布する際の onboarding 体験**を大きく改善します。`@cc-tealus` という技術的な mention を覚えなくても、AI として馴染みのある名前で呼べるためです。複数 project / 複数 alias の運用も、ファイル編集 + 再起動だけで完結します。

### HTTP transport — cross-machine 構成（Phase 1 alpha、[#264](https://github.com/gamasenninn/tealus/issues/264)） {#http-transport}

tealus-mcp は v0.12.0+ で **HTTP transport** を opt-in 提供しています。**Tealus 本体と Claude Code が別マシン** に居る cross-machine 構成（社内サーバ + 採用者ローカル PC 等）で、stdio が成立しない場合の構造解です。

| 経路 | Phase 1 alpha での扱い |
|---|---|
| **tools 呼び出し**（Claude Code → Tealus、Outbound） | HTTP 単独で動作（JWT auth、`/mcp` proxy 経由） |
| **mention 通知 wake-up**（Tealus → Claude Code、Inbound） | **stdio + file beacon 経路に依存**、HTTP では未サポート |

cross-machine 構成では **HTTP + cc-tealus stdio bridge 併用** が当面の現実解です。Phase 2 で SSE event broker（server-push wake-up）が乗ると HTTP 単独で完結する予定です。

- 詳細: [MCP Server > HTTP transport](mcp.md#http-transport)
- 採用者向け概念紹介: [Claude Code 連携](../guide/integration/claude-code.md)

### Phase B 展望

- **multi-session lock**（[#214](https://github.com/gamasenninn/tealus/issues/214)）: 別 PC で同 project を動かす場合の排他制御。現状は共有ストレージ（NAS / SMB / Syncthing）で `~/.tealus/cc-queue/` を mount する運用が想定されています
- **SSE event broker（HTTP transport Phase 2）**: server-push wake-up で HTTP 単独 cross-machine 構成を完結させる構造（[#264](https://github.com/gamasenninn/tealus/issues/264) Phase 2 候補）
- 他 dispatch 先の追加候補は本体 roadmap で議論中

## システムプロンプトのフロー {#system-prompt}

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

!!! info "Light v1 / Light v2 で `light_prompt.md` parity（v0.2.4〜、[#258](https://github.com/gamasenninn/tealus/issues/258) follow-up）"
    `{workspace}/light_prompt.md` は **Light v1 / Light v2 の両方で呼び出し毎に読み込まれ**ます（Light v1 と v2 の parity）。v0.2.4 以前の Light v2 は読み込んでおらず `/light2` 経由で system prompt が thin になる非対称がありましたが、v0.2.4 で fix されました。per-room MCP + per-room prompt の組み合わせで Light v2 でも query 精度を 1 室単位で tune 可能です。

### organon polyseme.sql_mapping auto inject pipeline（v0.3.0〜、[#276](https://github.com/gamasenninn/tealus/issues/276) follow-up） {#organon-polyseme-inject}

v0.3.0 (2026-05-31) で、**[organon](../concepts/organic-ontology.md) の `polyseme` data 層** から `sql_mapping` を持つ entries を抽出し、**全 agent (Light v1 / v2 / Deep / Codex Deep) の system prompt に自動 inject** する pipeline が追加されました。

| 仕組み | 詳細 |
|---|---|
| 共通 utility | `agent-server/src/lib/organonContext.mts`。polyseme entries → `sql_mapping` フィールド抽出 → prompt block 整形 |
| 注入 timing | **per-request `fs.readFileSync`**（cache なし）。organon 更新 → 次 request で即反映、**agent-server restart 不要** |
| inject 対象 | Light v1 / Light v2 / Deep / Codex Deep の **4 agent path 全て** |
| env | ★ **`ORGANON_INJECT=true` で有効（default OFF の opt-in）**、`ORGANON_REPO_PATH` で organon repo path override |
| graceful degrade | organon repo 不在環境では skip（稼働は継続）。ただし下記の warn あり |

!!! warning "env 名と既定値が変わりました（v0.4.0、[#304](https://github.com/gamasenninn/tealus/issues/304)）"
    **旧 `INJECT_ORGANON_POLYSEME`（default ON の opt-out）は廃止され、`ORGANON_INJECT`（default OFF の opt-in）になりました。互換 fallback はありません。**

    旧名のまま設定していると **何も注入されないまま静かに動作します**。organon を使う deployment では `ORGANON_INJECT=true` を明示してください。

    また v0.7.0（[#347](https://github.com/gamasenninn/tealus/issues/347)）から、`ORGANON_INJECT=true` なのに organon が見つからない / `sql_mapping` を持つ entries が 0 件の場合は、`[migration-check]` と同型の **loud WARN** が起動ログに出ます（「ON と宣言したのに何も注入されない」silent degrade の可視化。organon は任意サブシステムなので稼働は止めません）。

!!! info "inject 範囲の縮小（v0.7.0、[#349](https://github.com/gamasenninn/tealus/issues/349)）"
    以前は polyseme yaml を丸ごと注入していました（7 entry で 879 行）。DB 検索 grounding に必要な事実（table / field / 式 / filter / 混同注意）は `sql_mapping` ブロックに揃っており、その外の散文は grounding に marginal と**実機 A/B で検証**されたため、**`sql_mapping` ブロックのみに限定**されました（879 → 260 行、70% 減。grounding の結果は不変）。

この pipeline により、**organon-side が polyseme.sql_mapping を蓄積するだけで、全 AI agent が同時に「組織記憶辞書を読みながら」推論できる** architecture が成立しました。5/16 dogfood の **4 round LLM 訂正 → 1 round 正解** transition が両 agent path で N=2 evidence として再現されています。

詳細な architecture と 組織記憶辞書 = 動作する道具 narrative は [organic ontology 概念設計](../concepts/organic-ontology.md) を参照してください。

## 関連

- [AI アーキテクチャ](ai-architecture.md): 全体構造、Router の判定ロジック、Webhook Dispatcher、コスト設計
- [MCP Server](mcp.md): 18 ツールの詳細仕様と外部クライアントからの利用方法
- [AIエージェント設定](../admin/agent.md): 運用者向けの設定手順
- 関連 issue: [#185](https://github.com/gamasenninn/tealus/issues/185) MCP umbrella、[#194](https://github.com/gamasenninn/tealus/issues/194) search_messages、[#197](https://github.com/gamasenninn/tealus/issues/197) mark_tag_done、[#199](https://github.com/gamasenninn/tealus/issues/199) Light agent MCP 統合、[#200](https://github.com/gamasenninn/tealus/issues/200) create_room、[#207](https://github.com/gamasenninn/tealus/issues/207) delete_room、[#213](https://github.com/gamasenninn/tealus/issues/213) cc-tealus Phase A、[#214](https://github.com/gamasenninn/tealus/issues/214) multi-session lock（Phase B）、[#220](https://github.com/gamasenninn/tealus/issues/220) tealus-agent improvement harness（議論先行）、[#258](https://github.com/gamasenninn/tealus/issues/258) Light v2（codex SDK）、[#260](https://github.com/gamasenninn/tealus/issues/260) Light v2 機能 parity（send_text_as_file / generate_and_send_image）、[#263](https://github.com/gamasenninn/tealus/issues/263) cc-aliases.json
