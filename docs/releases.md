# リリースノート

## [Unreleased] — Phase 5 narrative emergence + 5/18 累積反映（2026年5月13-18日）

v0.2.4 cut（5/12）以降の累積。**5/17-5/18 で Phase 5 narrative（organic ontology / organon / feedback loop architecture）が docs surface 化**、**第 6 feedback layer = upstream pipeline rectification** が正典化、**organon Day 2 trace v0.5 release** で **vocab inject 第 1 例 feedback loop closure** が成立した転換点 release window です。

### Fixed

- **tealus-mcp `read_document` が `text/markdown` / `text/plain` / `text/csv` 等の text/* mime を扱えない問題を fix**（[#281](https://github.com/gamasenninn/tealus/issues/281)、5/18、tealus-mcp **v0.13.1** release `aa66826` + agent-server pin bump `5e89f64`）
    - 症状: Light agent (`/light`) が自作 markdown attachment を user 依頼で読み戻せず「この環境では本文の直接展開に失敗しました」と返す self-inflicted blind spot
    - 5/18 朝の朝礼ルームで再現: 動画 → 議事録 .md attach → user「内容表示」→ agent 読めず → user が手作業で再貼り付け（毎朝発生の可能性ありで urgency 高）
    - 真因: `documentReader.js` の `detectFormat` が PDF / DOCX / XLSX のみ判定、text/* は `unsupported` に倒れていた
    - 対応: `detectFormat` に `text` branch 追加（ext `md`/`txt`/`csv` または mime `text/*`）、`extractText` の switch に `case 'text'` 追加（`buffer.toString('utf8')` で本文返却、既存 `MAX_TEXT_LENGTH` で truncation）
    - tealus-mcp tests +4 / 全 95/95 pass、agent-server 3 pin（`lightV2.js` / `roomMcpManager.js` / `deep.js`）bump、production restart 後 朝礼で **1028 chars 本文 inline + TTS 8MB 読み上げ** で動作確認
    - ~40 分で 朝の調査 → Issue 起票 → TDD Min fix → release → pin bump → production verified まで完走、毎朝発生する trap の urgency を 1 cycle 内に消化した dogfood pattern

### Docs

- **[Tealus とは](what-is-tealus.md) 新規 — disclosure 階段の入口 doc**（5/18、本体 commit `cc3ed58`、147 行 / 9 section）
    - LP（controlled disclosure） / [organic ontology 概念設計](concepts/organic-ontology.md)（full disclosure） / organon（運用 manual）の間に位置する **入り口 doc** として、Tealus を初めて知る読者向けに「何か / なぜ / 何が起こるか / どう違うか / 本質 / 誰のためか / 始め方」を読者を選ばずに説明
    - LP narrative Layer 2（組織記憶）+ Layer 3（organic ontology）の middle layer として audience を絞り、詳細は 概念設計 / organon に降りる導線を提供
- **[organic ontology 概念設計](concepts/organic-ontology.md) を organon v0.5 / 第 6 feedback layer / vocab inject 第 1 例で update**（5/18、本体 commit `5825691`）
    - Layer 3: hazard family 8 → **11 軸**（whisper-prior / auto-minutes / stale-roster の 3 新軸）、organization namespace（v0.5 追加、初例 `44 = フォーティーフォー = 運送業者`）、設計原則を v0.5 時点に更新（identity-first verify 観点 5→8 拡張）、cover rate 表（4.55% → 27.3% → **57%**）+ entries 表（8 → 12 → 20）を追加
    - Layer 4: **Feedback loop architecture（6 段 layer system）** を新 subsection として追加、第 6 layer = upstream pipeline rectification = 外向き因果 loop の thesis 明記、organic ontology が形式論理自己言及 + Lakatos theory-ladenness を「内部 pragmatic 閉じ + 外部因果 loop 開き」の二段戦略で解消する整理を docs 物理化
    - 第 1 例（5/18 vocab inject 38→42）を Layer 4 に annotate、organon ↔ 本体 班 **23 分 round-trip closure** を AI 班連絡 channel で実現した経緯を記録
- **[アーキテクチャ](dev/architecture.md) Phase 4 中盤 構成図に SVG visual 化 追加**（5/18、本体 commit `208370a`）
    - `docs/images/phase4-architecture.svg` + `.png` 新規、3 段構造（Cloudflare+Nginx / 4 backend services / data stores + ChildProcess spawn）で技術構成を visual 化
    - 概念設計 doc で確立した SVG style（Noto Sans JP / Tealus brand teal / 結晶感配色）をアーキテクチャ doc にも適用、core 2（server / agent）を primary teal、extension 2（mcp / rtc）を light teal で差別化、外部 service（OpenAI / Aivis / Gemini / DALL-E / Anthropic）は bottom annotation で opt-in 明示
    - 既存 ASCII art は文字列 grep 可能な detail reference として残す（2 軸併用）
- **`docs/images/` 整備 — 3 SVG + 3 PNG が揃い、Phase 5 narrative visual 素材化の基盤完成**（5/18）
    - SVG: `organic-ontology-concept.svg`（center-out radial、5/17）/ `organic-ontology-architecture.svg`（4 層 trapezoidal、5/17）/ `phase4-architecture.svg`（3 段技術構成、5/18）
    - PNG（density 192 で 2x rasterize、slide deck / 外部共有 / SVG 描画不可な context 向け）
    - 概念図（radial）+ 概念構造（4 層）+ 技術構成（3 段）の **3 軸が同 style 統一**

### organic ontology / organon 連動

- **organon v0.5 release — Day 2 trace + Q&A 7 batch 結果反映**（5/18、organon repo commit `7aef27c`）
    - Q&A 7 件結果: **6/7 が誤変換 / 幻覚 / 過去職位、正解 1 件のみ** = identity 軸 family が systematic pipeline failure として class 化
    - entries 12 → **20 active**（67% 増）: 新規 confirmed role 7（神山 / 三瓶 / 舟太 / 草野部長 / 是枝 / 香山 / 齋藤）+ 新規 confirmed organization 1（44 = フォーティーフォー = 運送業者）+ 既訂正 1（桶田博信 + aliases [桶田専務, 専務] + stale roster note）
    - identity 軸 family 8 → **11 軸**: `#9 whisper-prior-misinference-to-vendor` / `#10 auto-minutes-stt-error` / `#11 stale-role-designation-in-roster`
    - hallucination 軸（#5）で 2 例目 = 再現性確立（みこがい農場の高山）、**organon が育つほど LLM の adversarial hallucination も高度化** 新仮説
    - schema v0.5: **organization kind 正式追加**（6 種類目 entry kind）、必須 field `vendor_class` + `not_in`
    - cover rate: 4.55% → 27.3% → **57%**（倍以上）= Phase A 拡張優先度の物的根拠 + 実装到達点
- **organon repo で第 6 feedback layer = upstream pipeline rectification を正典化**（5/18、organon repo `hazard-log/meta/upstream-pipeline-rectification-as-sixth-feedback-layer.md`）
    - 第 5 layer（内部 pragmatic 閉じ、評価対話）の **延長 + 補完戦略** として位置付け、外向き因果 loop で organic ontology が形式論理の自己言及問題 + Lakatos theory-ladenness 問題を **「内部 pragmatic 閉じ + 外部因果 loop 開き」の二段戦略** で解消する thesis 確立
    - organon CLAUDE.md の hazard 軸 table に **12 軸目** として正式 entry
- **第 1 例 feedback loop closure: organon hazard 発見 → 本体 STT pipeline 反映 → Day 3 朝礼で量的訂正効果測定 protocol**（5/18、本体班 ↔ organon 班 **23 分 round-trip** で完結、AI 班連絡 channel）
    - 本体 `server/config/transcription_guideline.json`（gitignored、per-deployment private）の vocabulary 38 → 42 entries 追加（神山 = 上山誤変換 / 三瓶 = アンプリ誤変換 / 舟太 = 中田くん誤変換 / 山崎整備長 = クラッチー 合成全体誤変換）、tealus 本体 server restart 後 effective
    - Day 3 朝礼 STT 出力で 4 件揺らぎ件数を Day 1〜2 と比較、量的訂正効果を organon Day 3 trace で measurement → v0.6 schema 設計 + 本体 linter MVP 着手の dep 解除 trigger とする protocol を organon side で commit
    - **distributed AI lane coordination**（organon ↔ 本体 班、user 介在ゼロ closure）が organic ontology の構造的必要条件であることが crystallize（[本体 Issue #279](https://github.com/gamasenninn/tealus/issues/279) (b) 一般理論の経験的根拠）

!!! info "5/13-5/15 の累積項目について"
    本 `[Unreleased]` section は **Issue #14 scope（5/18 累積反映）** に focus しています。5/13-5/15 にも複数の改善 commits（[#156](https://github.com/gamasenninn/tealus/issues/156) RoomSettings / [#273](https://github.com/gamasenninn/tealus/issues/273) remark-breaks / [#274](https://github.com/gamasenninn/tealus/issues/274) reply_to dispatch / iOS PWA 16px / [#168](https://github.com/gamasenninn/tealus/issues/168) 通知設定 / presentation docs 群）がありますが、本 release entry には未反映です。次回 drift audit cycle（次回 release tag cut タイミング想定）で集約予定。

## v0.2.4 — voice transcription completion + 動画文字起こし（2026年5月12日）

v0.2.3 cut（5/10）から **2 日 interval** での高速 release。Phase 4 中盤の「採用者第 2 号 不参加見込みの状況で逆説的に release pedagogy を磨く」観察期にあたる release で、**voice transcription pipeline を `gpt-4o-mini-transcribe` + 業務辞書 inject へ完成 phase**、**動画/音声文字起こし機能** （`transcribe_media` MCP tool）が追加されました。

### 新機能

- **動画/音声文字起こし** （[#271](https://github.com/gamasenninn/tealus/issues/271)、tealus-mcp v0.13.0）: `transcribe_media` MCP tool 新設。`get_message_media` の 10MB 上限で動画が取得できない問題を構造解、server-side で ffmpeg `-vn` audio 抽出 → Whisper API → AI 整形 → text 返却。`POST /api/bot/messages/:id/transcribe` 新 endpoint + thin MCP wrapper の組み合わせ。**MCP ツール 15 → 16**
- **PWA App Badge**（テスター要望、commit `6319cdd`）: ホーム画面アイコンに **未読数バッジ** を表示。Service Worker 経由で OS native badge API を叩き、Tealus を開かなくても未読件数が一目で分かる

### voice transcription 劇的改善（[#269](https://github.com/gamasenninn/tealus/issues/269) Phase 2 完走）

- **default model 切替**: `WHISPER_MODEL` default を `gpt-4o-transcribe` → **`gpt-4o-mini-transcribe`** に。採用者が `.env` 未設定でも最初から vocab inject 効果 + cost ~半分
- **model-aware vocabulary inject**: `WHISPER_VOCAB_INJECT_MODELS` env で対象 model を制御。`gpt-4o-mini-transcribe` / `gpt-4o-transcribe` に業務辞書を `用語: ...` 形式で whisper_context に注入、`whisper-1` は legacy bias 観測 history のため除外
- **prompt 上限拡張**: whisper-1 の 200 char (224 token 由来) から、新世代 transcribe の **2,000 char** (16,000 token の 1/8、安全側) に model-aware で切替
- **切り捨て方向 fix**: `slice(-N)` 末尾保持 → **`slice(0, N)` 先頭保持** に変更。whisper_context が先頭で安定、vocab list が末尾切れに
- **辞書 / aliases / guidelines 強化**: vocabulary 37 → 38 entries、aliases 35 → 46、業務固有 transcribe pattern 2 種追加 (`の→-` / `JA→D+日付-n`)
- **dogfood 6/6 完璧**: グレンコンテナ / マニアスプレッダ / ハーベスタ / みこがい / ガマ / たけのこ 等の業界用語 + 短人名すべて正確認識、control 文も clean、副作用なし

### Fixed — dormant bug 5 件

- **voice transcription 3 段 cascade fix** ([#269](https://github.com/gamasenninn/tealus/issues/269) Phase 2 follow-up): Whisper API が無音/ノイズ/短すぎる音声に対し **prompt を echo** する hallucination 検出 (`isWhisperPromptHallucination`)、短文 ( < 10 chars) を「意味なし」と判断する AI 整形 overreach の skip、「空文字」「(空)」「無音」等の **Japanese meta literal** を返す bug の post-process 防御 (`isMetaEmptyLiteral`)
- **light `agent_tool_end` heuristic 撤廃** (commit `3555cd3`): false positive で user に誤印象を与える「error」表示の判定 heuristic を撤廃
- **Deep agent timeout 構造 fix** ([#252](https://github.com/gamasenninn/tealus/issues/252) follow-up): timeout path に `deepRegistry.sweepByWorkspacePath()` 適用 (#252 cancel path と同型)、`setTimeout(..., 10000)` safety net で room queue dead lock 解消
- **ffmpeg Windows shell redirect**: 動画 audio 抽出時の Windows shell stderr 捕捉 + mp3 codec 統一 (`#271` follow-up)
- **agent-server tealus-mcp pin update**: pin を v0.11.1 → v0.13.0 に (Light v1 / v2 / Deep 全 agent path、`transcribe_media` を registered tools 一覧に追加)

### v0.2.3 後の累積 (5/10-5/11 完了済 work も含む)

v0.2.3 release tag cut (5/10) 後に completed していた以下も v0.2.4 に含まれます:

- **HTTP transport Phase 1 alpha** ([#264](https://github.com/gamasenninn/tealus/issues/264)、tealus-mcp v0.12.0-v0.12.3 の 4 release): cross-machine 構成への構造解 (下記 v0.12 group entry 参照、公開 docs 反映は [Issue #12](https://github.com/gamasenninn/tealus-docs/issues/12))
- **Light v1/v2 parity for `light_prompt.md`** ([#258](https://github.com/gamasenninn/tealus/issues/258) follow-up): Light v2 (`lightV2.js`) でも room workspace の `light_prompt.md` を読み込むよう parity 化、per-room MCP + per-room prompt の組み合わせで Light v2 でも query 精度を 1 室単位で tune 可能に
- **本体 docs 反映** ([#268](https://github.com/gamasenninn/tealus/issues/268)): `setup-ai-agent.md` に Light v2 + LIGHTV2_AUTH subscription path、`setup-cc-tealus-bridge.md` に cross-machine + Syncthing walkthrough + HTTP transport ステップ 5A、`03_アーキテクチャ設計.md` に Phase 4 narrative + HTTP transport 構成図 + Light v1/v2/Deep tier 表 を反映

!!! info "release pedagogy 観察"
    v0.2.3 (5/10) → v0.2.4 (5/12) の **2 日 interval** は Phase 4 中盤の高速 release pattern (1 日 5-9 commit の濃度) と整合します。採用者第 2 号 不参加見込みの状況で **release tag 自体が「voice transcribe が完成 phase に到達」という採用者向け narrative anchor** として機能している release です。

## tealus-mcp v0.12.0 - v0.12.3 — HTTP transport（cross-machine 構成）（2026年5月8-10日）

cross-machine MCP 構成への構造解として、**StreamableHTTPServerTransport** を導入（[tealus #264](https://github.com/gamasenninn/tealus/issues/264) Phase 1 alpha）。stdio path は default 維持で既存採用者環境は無変更、HTTP transport は **opt-in の選択肢**。

| version | 内容 |
|---|---|
| **v0.12.0**（5/8） | StreamableHTTPServerTransport 導入、初版 |
| **v0.12.1**（5/10） | dotenv で `.env` 読み込み追加（host mode 運用の友好度向上） |
| **v0.12.2**（5/10） | `/mcp/health` endpoint 追加（proxy 経由 reachability check） |
| **v0.12.3**（5/10） | **stateful session 管理**（proper MCP client = Claude Code 対応の構造 fix、`tools/list` failure を解消） |

tealus 本体側で `/mcp` proxy 追加（[tealus commit `b3fb3f7`](https://github.com/gamasenninn/tealus/commit/b3fb3f7) + pathRewrite fix [`6181ff0`](https://github.com/gamasenninn/tealus/commit/6181ff0)）、採用者向け walkthrough は本体 `setup-cc-tealus-bridge.md` ステップ 5A。

!!! info "公開 docs 側の HTTP transport 反映は別 issue で進行中"
    `dev/mcp.md` の HTTP transport section、`guide/integration/claude-code.md`（stdio default / HTTP optional の選び分け）、構成図などは本 release entry とは別の issue（Issue #12 候補）で反映予定。

## v0.2.3 — Phase 4 maturation（2026年5月10日）

5/3-5/10 の **Phase 4 中盤** で累積した 79 commits / 41 issues を整理した release。**Light v2（codex SDK backed）の並列追加**、tealus-mcp v0.10.0/v0.11.x 連携、PC 2-pane layout、Deep cancel path、E2E verification harness + LLM-as-judge layer、`cc-aliases.json` による `@Claude` alias 化、複数の採用者保護 trap 解消などを含む。

### Light v2 / agents

- **Light v2（codex SDK backed）並列追加**（[#258](https://github.com/gamasenninn/tealus/issues/258)）: `/light2` prefix、`@openai/codex-sdk` v0.128.0 経由、resident service ではなく **ephemeral subprocess via SDK** モデル。Light v1（OpenAI Agents SDK）は無変更で並列共存
- **`LIGHTV2_AUTH=subscription` 認証 path**（[#258](https://github.com/gamasenninn/tealus/issues/258) follow-up）: ChatGPT Plus / Pro 持ちの採用者は API cost 0 + Fast Mode access の選択肢。default は `api_key` で既存採用者を保護
- **router の mention 後 prefix 検出 fix**（[#258](https://github.com/gamasenninn/tealus/issues/258) follow-up）: group room の `@bot /light2 ...` が v1 落ちしていた bug、`stripLeadingMentions` helper 追加
- **agent_message multi-emit 修正**（[#260](https://github.com/gamasenninn/tealus/issues/260) follow-up）: codex SDK は 1 turn で `agent_message` を複数回 emit、最後の空文字列で前 text を上書きしていた bug。「最後の非空 agent_message を採用」に修正
- **tealus-mcp child process への env 伝播 fix**（[#260](https://github.com/gamasenninn/tealus/issues/260) follow-up）: codex SDK の `mcp_servers` config は `...process.env` 不継承、明示 env 指定で `OPENAI_API_KEY` / `GOOGLE_API_KEY` 等を child に渡す。副次的に [#261](https://github.com/gamasenninn/tealus/issues/261) vision fallback skip も解決

性能比較（実機 5/7 verify、gpt-4o-mini 単価）: 単純会話 v1/v2 2.07x cost / cross-room TODO 分類 v1/v2 3.78x cost、ただし cross-room 完結率は v2 が明確に勝る。推奨運用は **単純会話 / 単 room → v1、cross-room → v2、ChatGPT Plus 持ち → v2 (subscription path)**。

### MCP エコシステム拡張（ツール 11 → 15）

- **tealus-mcp v0.10.0**（5/5、[#254](https://github.com/gamasenninn/tealus/issues/254)）: `list_tags` ツール追加（bot 全 room の tag 一覧を usage 順で返す discovery primitive）、新 server endpoint `GET /api/bot/tags`。LLM 向け MCP は CRUD だけでなく **list / discovery primitive** が必須という教訓
- **tealus-mcp v0.11.0**（5/8、[#260](https://github.com/gamasenninn/tealus/issues/260)）: `send_text_as_file` + `generate_and_send_image` ツール追加（Light v2 機能 parity）。長文 text を file 添付 / DALL-E 3 経由の画像生成 + 投稿の composite action
- **tealus-mcp v0.11.1**（5/8、[#262](https://github.com/gamasenninn/tealus/issues/262) Phase 2 で surface）: `read_document` の `pdf-parse` parseError も Gemini Vision fallback の trigger 条件に追加（image-only PDF / 構造異常 PDF への robustness 向上）、README に env 名 trap 警告 callout（[#267](https://github.com/gamasenninn/tealus/issues/267)）
- v0.8.0 / v0.9.0 の `read_document` foundation（PDF / DOCX / XLSX 解析 + Gemini Vision fallback）も v0.2.3 期に統合

!!! warning "Gemini Vision fallback の Privacy 注意"
    Gemini free tier は Google が製品改善に利用、human reviewer が input/output を処理する可能性あり。社内文書を扱う場合は **paid billing account に紐付けた key 推奨**、または `DOCUMENT_VISION_PROVIDER=none` で disable。

### E2E verification harness + cc-aliases

- **E2E harness Phase 1**（[#262](https://github.com/gamasenninn/tealus/issues/262)）: `agent-server/tools/e2e/` に 6 scenarios、**3 layer judgment**（決定論層 / 観察層 / 人 review 層）。test bot user + test room で本番 DB 隔離、実機 path を全通す
- **LLM-as-judge layer（Phase 2.b）**（[#262](https://github.com/gamasenninn/tealus/issues/262)）: bot 応答を LLM（default `gpt-4o-mini`）に採点させる **観察層**（warn-only）。決定論層では捉えられない semantic correctness を可視化、score < min_score → warn (fail にはせず LLM 採点 variance 許容)
- **`cc-aliases.json`**（[#263](https://github.com/gamasenninn/tealus/issues/263)）: `@Claude` 等の自然な mention 名を Claude Code session に routing する設定ファイル化、code 変更不要で alias 追加可能

### 採用者保護（trap 連鎖解消）

- **rtc-server bundle.js auto-build**（[#234](https://github.com/gamasenninn/tealus/issues/234)）: `postinstall` で esbuild、起動時 sanity check で不在時 loud warn
- **test pollution 防止**（[#235](https://github.com/gamasenninn/tealus/issues/235)）: production code に `AGENT_CONFIG_DIR` / `AGENT_MCP_CONFIG_PATH` env override 追加、test 内で tmpDir に隔離。[#231](https://github.com/gamasenninn/tealus/issues/231) F1+F2 の表現を「admin UI 上書き耐性」→「test pollution 耐性」に訂正
- **`/system` redirect bug fix**（[#247](https://github.com/gamasenninn/tealus/issues/247)）: `app.get('/system/*', ...)` wildcard が `/system` 単独に match せず SPA fallback していた長期残留 bug
- **Router `max_completion_tokens`**（[#256](https://github.com/gamasenninn/tealus/issues/256)）: OpenAI 新 model で `max_tokens` が reject される問題
- **Vite dev server proxy 拡張**（[#257](https://github.com/gamasenninn/tealus/issues/257)）: `/agent-api` `/rtc` 漏れで dev mode の TTS / cancel / RTC が動かない trap、4 番目の採用者第 1 号 dogfood で発見
- **Portal iframe 通知 UX**（[#259](https://github.com/gamasenninn/tealus/issues/259)）: X-Frame-Options で iframe が空表示になる「ダンマリ」現象、常時表示の「↗ 新タブで開く」escape link + load timeout overlay
- **tealus-mcp env naming trap 警告**（[#267](https://github.com/gamasenninn/tealus/issues/267)）: `API_URL`（prefix なし）vs `TEALUS_API_URL` の silent fail 予防、README に ❌/✅ 比較 callout 追加
- **`setup-ai-agent.md` に Light v2 反映**（[#268](https://github.com/gamasenninn/tealus/issues/268)）: 採用者第 2 号 onboarding 直結、ステップ 9 で Light v2 / LIGHTV2_AUTH subscription path / 新 MCP tools 案内（本体 repo docs）
- **STT model 比較検証 Phase 1**（[#269](https://github.com/gamasenninn/tealus/issues/269)）: トランシーバー用途で `gpt-4o-mini-transcribe` + 辞書 prompt が業務無線音声で有望と判明、`WHISPER_MODEL` 用途別 guidance を `.env.example` に反映（コード改修 Phase 2 は v0.2.3 後で別途）

### UI / UX 完成度

- **`share_text_as_file` tool**（[#244](https://github.com/gamasenninn/tealus/issues/244)）: OCR / 整形 text を DL 可能 file として届ける agent custom tool。後に tealus-mcp v0.11.0 で `send_text_as_file` として MCP 化
- **hallucinated link 抑制**（[#245](https://github.com/gamasenninn/tealus/issues/245)）: gpt-5.4-mini が `sandbox:/mnt/data/...` を捏造する training bias、prompt-level で 2 layer 防御（tool description + `default_system_prompt.md` の禁止 URL section）
- **file 添付の DL filename を原本名に**（[#246](https://github.com/gamasenninn/tealus/issues/246)）: 旧 `<a target="_blank">` で cryptic basename になっていた問題を `<a download={file_name}>` で UX 完結。「ダウンロードは第一版要素」（user 観点）
- **文字起こし編集 modal の音声再生スライダー**（[#248](https://github.com/gamasenninn/tealus/issues/248)）: 原音を聴きながら文字起こしを修正できる、長尺の voice で効果的
- **Deep agent cancel**（[#250](https://github.com/gamasenninn/tealus/issues/250)-[#252](https://github.com/gamasenninn/tealus/issues/252)）: ユーザーが応答途中で中断できる UI button + Windows での `taskkill /T /F` + PowerShell sweep（[#252](https://github.com/gamasenninn/tealus/issues/252) で発覚した orphan kill の critical bug fix）+ redundant timeout message 抑制
- **mention picker に `cc-proj` 仮想 user**（[#253](https://github.com/gamasenninn/tealus/issues/253)）: `@cc-{project}` 形式を picker で選択可能に
- **Reply 引用 tap → 元 message scroll**（[#255](https://github.com/gamasenninn/tealus/issues/255)）: リプライ吹き出し内の引用部分タップで元のメッセージへスクロール + ハイライト

## v0.2.2 — cc-tealus + transcription quality jump（2026年5月2日）

業務メモから 5/2 朝に届いた現場声 3 件を 6 時間サイクルで Issue 起票 → 実装 → close まで完遂したリリース。Tealus の self-improving 哲学を release process そのもので体現。

- **cc-tealus リアルタイム連携 Phase A**（[#213](https://github.com/gamasenninn/tealus/issues/213)）: agent-server が `@cc-{project}` mention を検知して file beacon 経由で Claude Code session を sub-second wake-up。agent-server を「**AI 班 dispatcher**」へ進化させる構造判断の入口
- **voice 再文字起こし機能**（[#216](https://github.com/gamasenninn/tealus/issues/216)）: 文字起こしが失敗 / 不適切な voice メッセージを手動 retry 可能に。新 endpoint `POST /api/messages/:id/transcription/retranscribe`
- **gpt-4o-transcribe default 採用**（[#217](https://github.com/gamasenninn/tealus/issues/217)）: whisper-1 の hallucination（TV 字幕由来 noise 等）が大幅軽減。env `WHISPER_MODEL` で旧モデル戻し可
- **tealus-cli credentials defensive**（[#212](https://github.com/gamasenninn/tealus/issues/212)）: watch モードで JWT 再取得時の再発バグに対する起動時 freeze 防御層

## tealus-mcp v0.7.0 — get_messages verbosity 制御（2026年5月2日）

tealus 本体は `[Unreleased]` だが、tealus-mcp 側は初の versioned release。

- `get_messages` ツールに `include_transcription` / `include_raw` の 2 flag 追加（[tealus-mcp #1](https://github.com/gamasenninn/tealus-mcp/issues/1) / [tealus #219](https://github.com/gamasenninn/tealus/issues/219)）
- voice transcription inline 形式が 3 段階に: id-only / default（formatted_text のみ）/ +raw_text
- 本体側 consumer は `formatted_text || raw_text` fallback chain で動作、整形空 voice で raw garbage を LLM に渡さない improvement として動作

## v0.2.1 — install/onboarding patch（2026年5月1日）

v0.2.0 公開直後の他マシン install テストで発見された 2 件のバグを即日 fix した patch release。

!!! warning "Node.js 20+ 必須化（破壊的変更）"
    全 6 package.json に `engines.node: ">=20.0.0"`、ルート `.npmrc` に `engine-strict=true` を導入。**Node 18 環境では `npm install` が `EBADENGINE` で hard fail** します。原因は Node 18 に `File` global が未追加で `undici` v6+ が crash するためです。

- **Node.js 20+ 必須化**（[#210](https://github.com/gamasenninn/tealus/issues/210)）: cryptic crash → install 時点 hard fail に
- **初回ユーザー auto-promote**（[#211](https://github.com/gamasenninn/tealus/issues/211)）: 最初の非 Bot ユーザー（`is_bot=false` の COUNT が 0）が登録時に自動 admin role に。Mattermost / Rocket.Chat / GitLab 標準の OSS first-user-admin パターン

## v0.2.0 — OSS 公開直後の機能拡充（2026年4月30日）

v0.1.0 OSS 公開（2026-04-26）から 4 日間の累積成果。新機能 19 件 + behavior change 3 件 + bug fix 4 件。SemVer 0.x 慣習に従い minor バージョンアップ（v0.1.0 → v0.2.0）。

### MCP エコシステム拡張（11 ツール）

- **MCP 6 → 11 ツール**: `delete_room`（[#207](https://github.com/gamasenninn/tealus/issues/207)）/ `create_room`（[#200](https://github.com/gamasenninn/tealus/issues/200)）/ `search_messages`（[#194](https://github.com/gamasenninn/tealus/issues/194)）/ `mark_tag_done`（[#197](https://github.com/gamasenninn/tealus/issues/197)）/ `get_message_media`（[#185](https://github.com/gamasenninn/tealus/issues/185) 派生）
- **mcp-server 独立 repo 化**（[#187](https://github.com/gamasenninn/tealus/issues/187)）: [`gamasenninn/tealus-mcp`](https://github.com/gamasenninn/tealus-mcp)、`npx -y github:gamasenninn/tealus-mcp` で zero-config 利用
- **Light agent も Tealus MCP 統合**（[#199](https://github.com/gamasenninn/tealus/issues/199)）: 外部クライアントと本番 AI が同じ 11 ツールで動作

### Docker 全サービスデプロイ Phase A

- **`docker-compose.full.yml`**（[#188](https://github.com/gamasenninn/tealus/issues/188)）: `docker compose up` 1 コマンドで server / agent-server / client / dashboard を起動
- multi-stage build で client/dashboard dist 同梱、起動時マイグレーション自動実行
- Mac / Windows / Linux 全対応

### TTS 配信を rtc-server から独立

- **TTS Socket.IO blob default**（[#189](https://github.com/gamasenninn/tealus/issues/189)）: aivis-cloud TTS 配信経路を mediasoup → Socket.IO blob に切替、**rtc-server なしで Aivis 高品質 TTS 動作**
- legacy mediasoup 配信は `TTS_BROADCAST_MEDIASOUP=true` で復活可能（後方互換）

### voice transcription 自己改善基盤（内部実装）

- transcription pipeline カスタマイズ機構（[#204](https://github.com/gamasenninn/tealus/issues/204)）: vocabulary + guidelines を AI 整形段階に注入
- 自動学習 Phase 1 mining（[#206](https://github.com/gamasenninn/tealus/issues/206) / [#208](https://github.com/gamasenninn/tealus/issues/208)）: 編集履歴（AI 版 vs. 人間訂正版）から alias 候補を mining

### Pitch deck 公開

- **<https://tealus.dev/pitch/>**（[#209](https://github.com/gamasenninn/tealus/issues/209)）: Marp 形式 ~45 slides + 音声ナレーション、OSS 採用検討者向け Full pitch

### 後方互換性

True breaking change: **0 件**。すべての behavior change は env / config / migration で mitigation 可能。

注意点:
- migration 021 で `pg_trgm` extension 必要（managed PostgreSQL の場合は CREATE EXTENSION 権限要確認）
- `mcp-server` 旧 repo path から `github:gamasenninn/tealus-mcp` への切り替え推奨
- vite dev server で外部ホスト経由 access する場合 `VITE_ALLOWED_HOSTS` env 設定要

## v0.1.x — OSS 公開後の中間アップデート（2026年4月26日〜）

v0.1.0 OSS 公開後、Docker 化・TTS 配信刷新・MCP エコシステム拡張を中心に多数の改善が入りました。

### Docker フルデプロイ化（[#188](https://github.com/gamasenninn/tealus/issues/188)）

- `docker-compose.full.yml` で server / agent-server を 1 コマンド起動
- `docker-compose.rtc.yml` で rtc-server をオプショナル追加（Linux 限定）
- 各サービスに multi-stage Dockerfile（server ~312MB / agent-server ~261MB / rtc-server ~307MB）
- Mac / Windows / Linux 全対応（rtc-server 抜き default で host 制約なし）

### TTS 配信刷新（[#189](https://github.com/gamasenninn/tealus/issues/189)）

- 旧来の mediasoup PlainTransport 配信 → **Socket.IO blob 配信**を default に
- 結果: rtc-server なしで Aivis 高品質 TTS が動作可能に
- legacy 互換: `TTS_BROADCAST_MEDIASOUP=true` で旧経路に切替可能
- TTS 自動接続ロジック削除（[#190](https://github.com/gamasenninn/tealus/issues/190)）

### TTS 音量補正（[#198](https://github.com/gamasenninn/tealus/issues/198)）

- ユーザー単位の `voiceVolume`（Profile スライダー、`localStorage` 管理、0-100%）
- クライアントの組み込み倍率 `TTS_VOLUME_BOOST`（`client/src/constants/ui.js`、現在 3.0）
- Web Audio API GainNode で `<audio>.volume` の上限 1.0 を超える音量に対応

### MCP エコシステム拡張

- **MCP 独立 repo 化** ([#187](https://github.com/gamasenninn/tealus/issues/187)): `gamasenninn/tealus-mcp` を `npx -y github:gamasenninn/tealus-mcp` で zero-config 利用可能に
- **AI 画像視認** ([#185](https://github.com/gamasenninn/tealus/issues/185)): `get_message_media` で AI が画像を直接認識
- **全文検索 MCP** ([#194](https://github.com/gamasenninn/tealus/issues/194)): `search_messages` 追加、`pg_trgm` GIN index + UNION クエリで 70-80x 高速化
- **TODO 完了化 MCP** ([#197](https://github.com/gamasenninn/tealus/issues/197)): `mark_tag_done` でタグ `is_done` 切替
- **Light agent MCP 統合** ([#199](https://github.com/gamasenninn/tealus/issues/199)): Light/Deep agent が共通 Tealus MCP を共有、外部クライアントと本番 AI が同一ツールセットで動作

### UI 改善

- **Confirm 自前モーダル化** ([#191](https://github.com/gamasenninn/tealus/issues/191)): `window.confirm()` 全箇所を Zustand `confirmStore` + `<ConfirmModal />` に置換（ホスト名露出回避、ESC/Enter/overlay 対応）

### 環境変数の追加

- `TTS_BROADCAST_MEDIASOUP`（default `false`、legacy 互換用）
- agent-server の Bot 認証は `TEALUS_USER_ID` / `TEALUS_PASSWORD` を推奨（旧 `TEALUS_BOT_ID` / `TEALUS_BOT_PASS` も互換 fallback）

## v0.1.0 準備 — OSS 公開前リネーム（2026年4月24日）

!!! warning "破壊的変更"
    `employee_id`（社員番号）が `login_id`（ユーザーID）にリネームされました。DB カラム名、API フィールド名、JWT ペイロードがすべて変更されています。既存環境からのアップグレード時はマイグレーションが必要です。

- `users.employee_id` → `users.login_id`（DB カラム名）
- API リクエスト/レスポンスの `employee_id` → `login_id`
- UI 文字列「社員番号」→「ユーザーID」
- 環境変数名（`TEALUS_BOT_ID` 等）は変更なし

## Phase 4 — WebRTC 通話プラットフォーム（2026年4月18日〜）

- mediasoup SFU による音声・ビデオ通話
- グループ通話（複数参加者対応）
- 通話開始前の確認ダイアログ（誤タップ防止）
- Web トランシーバー（PTT: Push-to-Talk）
- 通話 UI 調整（ビデオミュート・グリッド改善・PiP）
- プッシュ通知（メッセージ受信・通話着信）

### TTS 読み上げ（音声合成）

- Aivis Cloud API による高品質日本語音声合成（レイテンシ 0.3〜0.5秒）
- AI 回答の自動読み上げ（agent-server 統合）
- トランシーバー自動接続/切断（ユーザー設定 ON/OFF）
- ルームごとの音声モデル選択（ダッシュボードから設定）
- 人気10モデルから選択可能（凛音エル、まお、にせ 等）
- 長文メッセージの折りたたみ表示（300文字超で「もっとみる」）

## Phase 3 — AI 連携（2026年4月6日〜）

- **リネーム**: Linny → Tealus
- CLI ツール（コマンドラインからメッセージ送信）
- VOX 音声連携（ディレクトリ監視モード）
- Webhook 機能
- AI エージェントシステム（Router + Light Agent + Deep Agent）
- Light Agent の OpenAI Agents SDK 移行
- Tealus MCP Server（Bot API を MCP ツールとして公開）
- システムダッシュボード
- エージェント設定（グローバル・ルーム単位）
- エージェント稼働モニタリング
- Deep Agent の Tealus MCP 統合
- TODO 機能（タグベース方式）
- Tealus Server ログ強化

## Phase 2 — 実用拡張（2026年4月3日〜）

- メッセージ検索（全ルーム横断・音声文字起こし含む）
- Bot API（テキスト・音声・画像・ファイル送信）
- CLI ツール（メッセージ送信・watchモード）
- メディアギャラリー + タグ機能
- すべて既読機能
- 既読システムをカーソル方式に変更
- AI スタンプ生成 + スタンプパック

## Phase 1 リファクタリング（2026年4月3日）

- メンバーシップチェックの共通ミドルウェア化
- エラーメッセージの定数化
- Socket.IO ハンドラの分割
- ChatRoom のカスタムフック抽出
- CSS 共通化・CSS 変数化
- 130テスト pass 確認

## Phase 1 — MVP（2026年3月27日〜）

- プロジェクト基盤セットアップ（Docker Compose + PostgreSQL + Redis）
- 認証（ユーザーIDログイン・JWT）
- ルーム管理（1対1・グループ）
- メッセージ送受信（REST API + WebSocket）
- メディアアップロード（画像・動画・ファイル・サムネイル自動生成）
- 既読管理（未読数・既読数・リアルタイム更新）
- リプライ（引用返信）
- Push 通知（Service Worker + Web Push）
- LINEライク UI（吹き出し・PWA）
- 複数画像グリッド表示・画像ビューア
- 音声メッセージ（録音・Whisper 文字起こし・AI 整形・編集履歴）
- ユーザー管理画面（管理者ダッシュボード）
- ユーザープロフィール編集
- グループメンバー管理（追加・退会・除外・管理者変更）
- 文字サイズ設定
- メッセージ削除・コンテキストメニュー
- 日付区切り・通知音
- 絵文字リアクション
- 単独グループ（メモ機能）
- ブランドカラー変更（LINE緑 → ティール）
- PWA ビルド・本番配信設定
