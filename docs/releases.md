# リリースノート

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
