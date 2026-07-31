# リリースノート

このページは **「使う人」向けの要約**です。各リリースで何が変わったか、そして**更新するときに何をしないと壊れるか**を短く載せます。

- **実装の詳細と設計判断の理由** → 本体リポジトリの [CHANGELOG](https://github.com/gamasenninn/tealus/blob/main/CHANGELOG.md)
- **自己ホスト環境の具体的な更新手順** → 本体リポジトリの [アップグレードガイド](https://github.com/gamasenninn/tealus/blob/main/docs/upgrade-guide.md)

!!! note "このページの形式について"
    以前は本体 CHANGELOG とほぼ同じ内容を二重に持っていましたが、**二重管理は必ず片方が腐り、腐ったときに読者がどちらを信じてよいか判断できません**。そのため詳細は CHANGELOG に一元化し、このページは要約とリンクに絞る方針へ切り替えました。

    **v0.4.0 以降**がこの新しい形式です。**v0.3.0 以前**は当時の詳細な記述をそのまま残しています。また、新しい機能はガイドページの整備が追いついていないものがあります（[tealus-docs#17](https://github.com/gamasenninn/tealus-docs/issues/17) で対応中）。

## v0.7.0 — 出したものが端末に届き、使うほど指示が短くなる（2026年7月31日）

iOS の PWA に更新が届かない構造的な問題を根治し、あわせてエージェントへの指示を「登録」させず過去の自分の投稿から再利用できるようにしたリリースです。

**主な変更**

- **PWA 更新検知**（[#355](https://github.com/gamasenninn/tealus/issues/355) / [#356](https://github.com/gamasenninn/tealus/issues/356)）— 新しいクライアントが自分の陳腐化に気づいて更新バナーを出すようになりました。**プロフィール画面の下部にビルド ID が常時表示**され、実機が新旧どちらかを目視で判定できます
- **エージェント指示の履歴ピッカー**（[#354](https://github.com/gamasenninn/tealus/issues/354) / [#357](https://github.com/gamasenninn/tealus/issues/357) / [#358](https://github.com/gamasenninn/tealus/issues/358)）— 🤖 パネルに「最近の指示」が出ます。登録は不要で、過去に自分が送った `@アシスタント` / `@cc-*` 宛のメッセージをそのまま再利用できます。PC では `/` → `↑` がシェルの履歴と同じ操作です。「直近の画像 N 枚で〜」のように数字だけが変わる指示は 1 件にまとまり、**変わる位置だけが選択状態**で挿入されます（挿入するだけで、送信はしません）
- **ルームのタグ削除**（[#351](https://github.com/gamasenninn/tealus/issues/351)）— 従来は追加のみでしたが、タグ画面の「管理」トグルから削除できるようになりました
- **LINE の複数画像を 1 メッセージにまとめる**（[#353](https://github.com/gamasenninn/tealus/issues/353)）— 順不同で別々に届く画像を待ち合わせて 1 件に再構成します（待ち時間は `LINE_IMAGESET_FLUSH_MS`、既定 15 秒）。詳細は [LINE Bridge 連携](guide/integration/line-bridge.md)
- **organon なしでも語彙の正規化が効く**（[#348](https://github.com/gamasenninn/tealus/issues/348) / [#349](https://github.com/gamasenninn/tealus/issues/349)）— 辞書テーブルから `dictionary.local.ttl` を発行し、エージェントはこれを優先して読みます。organon を導入していない環境でも別名→正規名の正規化が生き続けます

!!! warning "★ 更新時の急所"
    - **DB migration の追加はゼロ**です
    - ★ `cd client && npm run build` — **再ビルドしないと更新検知そのものが入りません**（エラーにはならず、静かに機能しません）
    - ★ `cd agent-server && npm install` + 再起動 — 依存に `n3` が追加されました。忘れると **agent-server が起動時に落ちます**
    - ★ **この版に上げるときだけ**、各端末で 1 回だけ手動リロードが必要です（旧クライアント側に更新を検知する手段がないため）。**次回以降はバナーが自動で出ます**
    - リバースプロキシを挟んでいる場合、`index.html` / `sw.js` / `*.webmanifest` の Cache-Control をプロキシ側が上書きしていないか確認してください

[変更履歴 v0.6.0...v0.7.0（19 コミット）](https://github.com/gamasenninn/tealus/compare/v0.6.0...v0.7.0) ／ [リリースページ](https://github.com/gamasenninn/tealus/releases/tag/v0.7.0)

## v0.6.0 — AI に「構造」を渡す（2026年7月21日）

AI に定型の確認や判断を集めさせるための**汎用フォーム**と、「アシスタントに聞いたつもりが無反応だった」を減らす**エージェントの起動入口**が入りました。

**主な変更**

- **汎用フォーム**（[#336](https://github.com/gamasenninn/tealus/issues/336)）— ルームに回答フォーム（単一選択 + 自由記述）を投稿できます。回答者が [回答する] を押すと、回答が返信として投稿されると同時に担当エージェントが起動します（コピペ + 手書き + 別途メンション、が 1 ボタンに畳まれます）。MCP からは `send_form`（[MCP Server](dev/mcp.md)）
- **エージェント起動入口**（[#338](https://github.com/gamasenninn/tealus/issues/338)）— 入力欄の 🤖 ボタン / メッセージ長押しの「エージェントに送る」で正しい宛先が自動で入ります。**呼び忘れたメッセージを編集して先頭にメンションを足すと、そこから起動**します
- **未適用 migration の起動時 warn**（[#334](https://github.com/gamasenninn/tealus/issues/334)、採用者保護）— migration を忘れた状態でも起動してしまい、機能だけが静かに欠ける事故を、起動ログの `[migration-check]` で可視化しました
- **エージェント受付の可視化**（[#335](https://github.com/gamasenninn/tealus/issues/335)）— `@cc-*` を投げたときに「届きました」と表示が出るようになり、「反応が返るのか分からないまま待つ」状態が解消しました

**直った不具合** — スリープ復帰後に「考え中 / 入力中」が残る（[#340](https://github.com/gamasenninn/tealus/issues/340)）／フォームの回答ボタンが何度も押せる二重回答（[#341](https://github.com/gamasenninn/tealus/issues/341)）

!!! warning "★ 更新時の急所"
    - ★ `cd server && npm run migrate` — 追加は `026_message_form_type.sql` の 1 本。**忘れるとフォーム投稿が DB 制約で失敗**します
    - ★ `cd client && npm run build` — フォーム UI と起動入口が入っているため再ビルド必須
    - `cd agent-server && npm install` + 再起動
    - MCP から `send_form` を使う場合は **tealus-mcp v0.14.8 以上**（タグ固定運用なら pin 更新 + Claude Code のフル再起動）。本体の更新だけでは MCP クライアント側は変わりません

[変更履歴 v0.5.0...v0.6.0（16 コミット）](https://github.com/gamasenninn/tealus/compare/v0.5.0...v0.6.0) ／ [リリースページ](https://github.com/gamasenninn/tealus/releases/tag/v0.6.0)

## v0.5.0 — TypeScript 化と、自己成長する辞書（2026年7月14日）

コードベース全体の TypeScript 化という土台の入れ替えと、**人の修正履歴から別名を自動で学習していく辞書**が入った、範囲の大きいリリースです。

**主な変更**

- **コードベース 100% TypeScript 化**（[#330](https://github.com/gamasenninn/tealus/issues/330)）— 全レイヤーを `.mts` / `.ts` 化し、サーバーは**ビルドせず直接実行**する構成になりました（型検査は別途 `tsc --noEmit`）。★ **この変更により Node 24 が必要**です（[サーバー構築](admin/server-setup.md)）
- **自己成長する変換辞書**（[#327](https://github.com/gamasenninn/tealus/issues/327)）— 別名→正規名の辞書がファイルから DB テーブルへ移りました。**人が文字起こしを直した履歴から別名を自動学習**し、精度の基準を満たしたものだけが実際に効き始めます。管理ダッシュボードに承認 / 却下の育成画面が付きました（[ダッシュボード](admin/dashboard.md)）
- **自ホスト STT + organon 文脈補正**（[#326](https://github.com/gamasenninn/tealus/issues/326)）— `TRANSCRIPTION_MODE=organon` で、自ホストの音声認識の出力を組織の語彙で補正する 2 段構成に切り替えられます（データ主権を保ったまま固有名詞を回収する構成）
- **organon から辞書を自動取り込み**（[#331](https://github.com/gamasenninn/tealus/issues/331)）— organon 側の更新が 5 分以内に辞書と文字起こしへ自動反映されます
- **画像・帳票まわりの強化** — 貼り付け（Ctrl+V）でのアップロード（[#311](https://github.com/gamasenninn/tealus/issues/311)）／1 メッセージの**複数画像・複数文書をすべて読めるように**（[#316](https://github.com/gamasenninn/tealus/issues/316) / [#317](https://github.com/gamasenninn/tealus/issues/317)、従来は 1 件目だけ）／業務語彙を画像 OCR にも効かせる `VOCAB_INJECT`（[#315](https://github.com/gamasenninn/tealus/issues/315)、既定 OFF）
- **完了を表す ✅ リアクションと、その抽出**（[#323](https://github.com/gamasenninn/tealus/issues/323) / [#325](https://github.com/gamasenninn/tealus/issues/325)）— 「✅ が付いていないもの一覧」をエージェントが確実に絞り込めるようになりました
- **Deep のタイムアウトを 5 分 → 8 分に延長**（[#314](https://github.com/gamasenninn/tealus/issues/314)）

**直った不具合** — 音声認識が日本語を中国語と誤検出して文字起こしが壊れる（[#332](https://github.com/gamasenninn/tealus/issues/332)）／日本語ファイル名の文字化け（[#319](https://github.com/gamasenninn/tealus/issues/319)）／画像生成が全失敗（[#313](https://github.com/gamasenninn/tealus/issues/313)、`gpt-image-1` へ移行）／「ルーム情報の取得に失敗しました」の頻発（[#322](https://github.com/gamasenninn/tealus/issues/322)、通信の間欠停止に対するリトライ）

!!! warning "★ 更新時の急所"
    - ★ **Node 24 が必要です。** TypeScript 化により、型注釈をそのまま解釈できない版では起動しません。24 未満では `npm install` 自体が `EBADENGINE` で止まります（[サーバー構築 > 前提条件](admin/server-setup.md)）
    - ★ `cd server && npm run migrate` — 辞書テーブルの `023`〜`025` が追加されました。**忘れると辞書育成タブが 500 になり、自己成長辞書が静かに効きません**（この事故を可視化する起動時 warn は次の v0.6.0 で入りました）
    - `cd client && npm run build` + 全パッケージの `npm install` と再起動

    このリリースはアップグレードガイド（v0.6.0 で新設）のバージョン別ノートより前のため、上記は CHANGELOG からの要約です。

[変更履歴 v0.4.0...v0.5.0](https://github.com/gamasenninn/tealus/compare/v0.4.0...v0.5.0) ／ [リリースページ](https://github.com/gamasenninn/tealus/releases/tag/v0.5.0)

## v0.4.0 — 外部チャネル連携と、room を越える AI 協働（2026年6月21日）

LINE からの投稿を取り込む外部連携と、AI が room の境界を越えて協働するための明示構文が入りました。

**主な変更**

- **LINE Bridge（受信）**（[#288](https://github.com/gamasenninn/tealus/issues/288) 〜 [#291](https://github.com/gamasenninn/tealus/issues/291) / [#309](https://github.com/gamasenninn/tealus/issues/309)）— LINE 公式アカウントへの投稿を Tealus のルームに取り込みます。テキスト / 画像 / 音声（自動で文字起こし）/ 動画 / ファイル / スタンプ / 位置情報の 7 種に対応。詳細は [LINE Bridge 連携](guide/integration/line-bridge.md)
- **`%委譲%` — room 間の AI 委譲**（[#295](https://github.com/gamasenninn/tealus/issues/295) / [#292](https://github.com/gamasenninn/tealus/issues/292)）— 「どう頼んだか」の言い回しに依存せず、明示構文で確実に他ルームの AI へ委譲できます。複数ルームへの同時委譲も可能です。詳細は [Agent ガイド](dev/agents.md#delegation-syntax)
- **Light の backend を設定で切り替え可能に**（[#292](https://github.com/gamasenninn/tealus/issues/292)）— `AGENT_LIGHT_BACKEND` で v1 / v2 を選べるようになり、**既定が v2 に変わりました**。詳細は [Agent ガイド](dev/agents.md#light-v1-v2)
- **アクセスログ**（[#301](https://github.com/gamasenninn/tealus/issues/301)）— 管理ダッシュボードに最終投稿 / 最終閲覧の集計ビュー。詳細は [ダッシュボード](admin/dashboard.md)
- **ゲスト role の運用開始**（[#282](https://github.com/gamasenninn/tealus/issues/282)）— 外部の問い合わせ者向けに、見える範囲を絞った役割を運用可能にしました。詳細は [AIエージェント設定](admin/agent.md)
- **メッセージの部分コピー**（[#308](https://github.com/gamasenninn/tealus/issues/308)）／**文字起こしの連続編集**（[#302](https://github.com/gamasenninn/tealus/issues/302)、前後の音声へ送りながら編集）／**長文 TTS の失敗防止**（[#300](https://github.com/gamasenninn/tealus/issues/300)、[TTS 読み上げ](guide/advanced/tts.md)）

**直った不具合** — 「考え中」表示が残り続ける / 他ルームに漏れる（[#298](https://github.com/gamasenninn/tealus/issues/298) / [#297](https://github.com/gamasenninn/tealus/issues/297)）／1 件の描画エラーでチャット全体が白紙化する（[#306](https://github.com/gamasenninn/tealus/issues/306)）／送信に失敗しても「送信済み」に見える（[#303](https://github.com/gamasenninn/tealus/issues/303)）／Codex の認証トークンが繰り返し失効する（[#307](https://github.com/gamasenninn/tealus/issues/307)）

!!! warning "★ 更新時の急所"
    - **DB migration の追加はゼロ**です
    - ★ **organon を使っている場合、環境変数の名前と既定値が変わりました**（[#304](https://github.com/gamasenninn/tealus/issues/304)）。`INJECT_ORGANON_POLYSEME`（既定 ON）は**廃止**され、`ORGANON_INJECT=true`（**既定 OFF の opt-in**）になりました。互換の読み替えはないため、**旧名のままでは何も注入されずに静かに動きます**
    - ★ **Light の既定 backend が v1 → v2 に変わりました。** 従来どおり v1 で運用したい場合は `AGENT_LIGHT_BACKEND=v1` を明示してください
    - `cd client && npm run build` + 再起動

    このリリースはアップグレードガイド（v0.6.0 で新設）のバージョン別ノートより前のため、上記は CHANGELOG からの要約です。

[変更履歴 v0.3.0...v0.4.0](https://github.com/gamasenninn/tealus/compare/v0.3.0...v0.4.0) ／ [リリースページ](https://github.com/gamasenninn/tealus/releases/tag/v0.4.0)

## v0.3.0 — 「組織記憶辞書が動作する道具に転化した release」（2026年6月1日）

Phase 4 **organon paradigm operational realization** 完成。Day 14-15-16（5/30-6/1）で **Codex Deep agent + organon polyseme inject pipeline + 5/16 ground truth pattern 解消 + 4 Issue cluster 連続 close** を達成、organon が業務 DB を upstream rectification する thesis が **structural に完成** した転換点 release。

5/12 v0.2.4 → 5/19 organon Day 3 量的成功 → 5/30 Day 14 organon 日次サイクル workflow 化 → 5/31 Day 15 Codex Deep + polyseme inject pipeline 完成 → 6/1 Day 16 release cut の **20 日間の進化** を集約。

### Codex Deep agent + Light backend config 化 + cross-room delegation

- **Codex Deep agent (`deepCodex.js`) 追加 + `DEEP_AGENT_PROVIDER` で切替**（[#276](https://github.com/gamasenninn/tealus/issues/276)、5/31 Day 15、commit `41b0f55`）: 既存 Deep agent（`claude -p` spawn）と並列で `codex exec` spawn 経路を追加。Light v2 で確立した Codex SDK / subscription auth pattern を CLI mode で再現
    - env `DEEP_AGENT_PROVIDER=claude|codex`（default=claude、既存維持）で switch
    - env `DEEP_CODEX_AUTH=subscription|api-key`（default=subscription）で `~/.codex/auth.json` 経由 ChatGPT subscription 認証 = **API cost 0**（Deep mode の 12-50k tokens × call 課金爆発 risk 回避 default）
    - `CODEX_HOME` 動的切替（workspace 配下 `.codex_home/`）、`AGENTS.md`（= `CLAUDE.md` literal copy、Codex CLI 自動 read project context、上限 32 KB）
    - TDD test 34 件追加、全 305 → 317 Green、5/31 dogfood 第 1 例 success（subscription auth confirmed）
    - **Claude MAX 契約必須を超えた選択肢拡大、Anthropic / OpenAI 両 provider Deep mode 同等運用可能化**。詳細は [Agent ガイド > Deep agent — provider 切替](dev/agents.md#deep-provider)
- **AGENTS.md (project root) 新規** = `CLAUDE.md` literal copy（[#276](https://github.com/gamasenninn/tealus/issues/276)、5/31 commit `41b0f55`）

### organon polyseme.sql_mapping auto inject pipeline

- **agent-server: organon polyseme.sql_mapping を全 agent system prompt に自動 inject pipeline**（[#276 follow-up](https://github.com/gamasenninn/tealus/issues/276)、5/31 Day 15、commit `8f28279`）
    - 共通 utility `agent-server/src/lib/organonContext.js` 新規追加、polyseme entries から `sql_mapping` 持つ entries 抽出 + prompt block 整形、**per-request `fs.readFileSync`**（cache なし）で organon 更新即時反映
    - 全 4 agent（Light v1 / v2 / Deep / Deep Codex）の system prompt build に `loadOrganonPolysemeForPrompt()` を 1 行 inject
    - env `INJECT_ORGANON_POLYSEME=true`（default）で inject 有効、`ORGANON_REPO_PATH` で organon repo path override、organon repo 不在環境 silent skip
        - ★ **現在は無効な記述です（当時の仕様）**: v0.4.0（[#304](https://github.com/gamasenninn/tealus/issues/304)）で **`ORGANON_INJECT`（default OFF の opt-in）** にリネームされ、旧名は廃止（fallback 無し）。現行仕様は [Agent ガイド](dev/agents.md#organon-polyseme-inject) を参照
    - **agent-server restart 不要**（organon entries 更新 → 次 request で即反映）、organon-side cycle 1 度回せば 全 agent operational value 即享受
    - 5/31 dogfood 第 2 例 success: Deep Codex 5818 chars / Light system prompt 31756 chars、**両 agent で 5/16 4 round → 0 round 正解 transition N=2 evidence**、user 訂正 0 round
    - 詳細は [Agent ガイド > organon polyseme.sql_mapping auto inject pipeline](dev/agents.md#organon-polyseme-inject)
    - **構築 → data → middleware → 利用 → 業務 自動 pipeline 完成、organon-side が polyseme.sql_mapping を蓄積するだけで全 AI agent が同時に賢くなる architecture realized**

### 採用者保護 + ゲストユーザ Phase 1 MVP

- **tealus-mcp `read_document` が `text/markdown` / `text/plain` / `text/csv` 等の text/* mime を扱えない問題を fix**（[#281](https://github.com/gamasenninn/tealus/issues/281)、5/18、tealus-mcp **v0.13.1** release `aa66826`）: Light agent が自作 markdown attachment を読み戻せない self-inflicted blind spot を fix（5/18 朝の朝礼ルームで再現 → ~40 分で release + production verified）
- **ゲストユーザ role 拡張 Phase 1 MVP**（[#282](https://github.com/gamasenninn/tealus/issues/282)、5/19、3 commits: `b986bad` + `956fc9e` + `a7a102a`）: Issue 起票 → TDD 実装 → production deploy までを **1 day で完走**
    - **Phase A**: migration 022 で `users.role` CHECK constraint 拡張（`('admin', 'user')` → `('admin', 'user', 'guest')`）、既存 user data 影響なし
    - **Phase B**: `server/src/utils/permissions.js` 新規、`isAdmin/isUser/isGuest` role-class check + `canCreateRoom/canInviteToRoom/canSearchUsers` ability check、既存 admin チェック 5 箇所を helper 関数経由に統一
    - **Phase C**: route-level access control（POST /api/rooms / GET /api/users 等に guest guard、guest → 403）
    - 累計 **+33 test cases / 全 38 suite / 414 tests pass、regression ゼロ**
    - 3 層構造の正典化（Layer 1 schema / Layer 2 logic / Layer 3 route guard）、[#124](https://github.com/gamasenninn/tealus/issues/124) ゲストルーム構想からの pivot、採用者 #2 voice trigger 解除 candidate
- **migration 016 announcement 誤公開 bug 修正**（5/21、commit `b535dca`）: `npm run migrate` 再実行のたびに `is_published=true` 書き戻しが走る bug を fix、`recovery-2026-05-21-announcement-republish-fix.sql` を提供
- **#287 vocab cache reset endpoint**（`POST /api/admin/transcription/reload-vocab`、5/28、commit `12d67cf`）: `transcription_guideline.json` 編集後の即時反映用、server restart 不要

### docs surface 化 — Phase 5 narrative の docs 化

- **[Tealus とは](what-is-tealus.md) 新規 — disclosure 階段の入口 doc**（5/18、本体 commit `cc3ed58`、147 行 / 9 section）: LP（controlled disclosure）/ [organic ontology 概念設計](concepts/organic-ontology.md)（full disclosure）/ organon（運用 manual）の middle layer
- **[organic ontology 概念設計](concepts/organic-ontology.md) 大幅 update**: organon v0.5 → v0.5.1 → v0.6 schema candidate（cover rate 4.55% → 27.3% → 57% → 74%、entries 8 → 12 → 20 → 26、hazard family 8 → 11 → 24 軸）、Layer 4 で **第 6 layer sub-pipeline 6a + 6b 構造**（SQL bridge thesis、5/21）+ **第 7 layer candidate**（architect-mediated organon ingestion、5/19 + 5/21 で N=2 near-confirmed）+ **14 軸目 candidate**（observer-architect-duality、N=2、5/19 + 5/21）+ maturation curve（5 日連続 layer surface、5/17-5/21）+ **3 段機能進化**（観測装置 → 自己訂正装置 → 元データ整流装置）の Stage 3 外向き因果 loop が artificial ontology との分水嶺
- **第 1 例 feedback loop 量的成功 4/4 ✅ confirmed**（5/19 Day 3 朝礼 STT）: 「上山」「アンプリ」「中田」「クラッチー」全て 0 件、27 時間で round-trip closure 完結（organon Day 2 hazard 発見 → 23 分 round-trip で本体 vocab 38→42 → Day 3 量的確認）
- **[アーキテクチャ](dev/architecture.md) に Phase 4 中盤 SVG + Tealus ↔ organon 接続層 SVG 追加**（5/18 + 5/30、本体 commits `208370a` + `97a9976`）
- **`docs/images/` 整備**: organic-ontology-concept.svg / organic-ontology-architecture.svg / phase4-architecture.svg / organon-functional-evolution.svg / tealus-organon-architecture.svg（5 SVG）+ 同名 PNG（2x rasterize）、Noto Sans JP / Tealus brand teal / 結晶感配色の 統一 style
- **organon repo daily cycle workflow 化**（Day 14、5/30、[#286](https://github.com/gamasenninn/tealus/issues/286) close）: `.claude/skills/organon-daily/SKILL.md` 構築層 skill 化境界設計
- **4 Issue cluster 連続 close**（5/30-6/1）: [#286](https://github.com/gamasenninn/tealus/issues/286)（organon 日次サイクル）/ [#276](https://github.com/gamasenninn/tealus/issues/276)（Codex Deep + spike）/ [#283](https://github.com/gamasenninn/tealus/issues/283)（organic ontology × DB bridge = 6b SQL upstream rectification）/ [#202](https://github.com/gamasenninn/tealus/issues/202)（AI 協業基盤 / 班間情報非対称 umbrella、structural completion marker）

### UI + presentation docs（5/13-5/15 累積も統合）

- **`#284` STT default → `gpt-4o-mini-transcribe`**（5/25、commit `58daf43`）: 無音/ノイズ幻覚を既存 filter で空化、組織記憶辞書 + 業務無線 use case で安定
- **`#156` Phase 1: チャット画面 RoomSettings に「エージェント設定」section + TTS 音声モデル選択**（5/15 + 5/25、commits `6ac122f` + `a0b0e24`）: ルームメニュー → 設定 から応答モード（auto / all / mention / off）と Light Agent / Deep Agent prompt + TTS 音声モデルを変更可能。`#156` follow-up（5/15、commit `b3f4ec9`）で RoomSettings section 順序を「個人 → ルーム（admin）→ システム（sysAdmin）→ エージェント」に変更
- **`#273` メッセージ表示で段落内改行 `\n` を `<br>` として render**（5/14、commit `bdf3ccc`、`remark-breaks` plugin 追加）: CommonMark soft wrap で消えていた user 入力改行を保持、コードブロック / MD 強調 / table / list は完全 unchanged
- **`#274` agent prompt に `reply_to` / `reply_to_message` を伝達する構造修正**（5/14、commits `28698bb` + `c8f044b`）: L1 server webhook payload + L2 agent-server dispatcher（`buildReplyToHint` helper）+ L3 room config の 3 層、議事録 9 section 100% 反映を dogfood verify
- **`#208` follow-up: tealus-mcp v0.14.0 `get_message_edit_history` MCP tool**（5/24、commits `c8afcb4` + `1c85073` 等）: text edits + voice/video transcription versions 統合返却、organon daily cycle の edit history 観察用。**MCP ツール 16 → 17**。5/24 Day 8 user **1 instrument 主義**提案起源、mining script の正式 deprecation 候補（Day 10 で確定）
- **iOS PWA input focus 時の auto-zoom 防止**（5/13、commit `4d00839`）: iOS Safari の auto-zoom 挙動（font-size < 16px の input にフォーカスすると画面拡大）を採用検討者 iPhone dogfood で観察 → 翌日 fix
- **MemberList のメンバー操作 button をメンバー一覧直下に移動**（5/15、commit `1f849f4`）: Gestalt 近接の原則準拠、`#156` で UX 全体が見えて気付いた連鎖改善
- **`#168` Phase 1 sub-task: 通知設定 doc + トラブルシューティング新規作成**（commit `497b5fb`）: 5 section 構成（PWA install / 通知許可 / バッテリー最適化 / 二経路 defense in depth / 既知制限）
- **`#209` sub-1.1: `philosophy.md` v2 — organic ontology section 新設**（commit `4c81236`）: 5/11 user 言語化（organic ontology）を 4 柱の手前に layer として配置、5/11-5/14 dogfood の cycle 検証 4 例
- **`#209` sub-1.2: `elevator-pitches.md` v2 — 推奨案 E に 3 軸追記**（commit `a3490bc`）: 業務 DB / cross-modality dividend / 採用者 voice → 機能進化 cycle
- **`#209` sub-2: README opening narrative refresh + ロードマップ v0.2.x 反映**（commit `a9a3444`）: tagline「AI が組織の記憶を声で運ぶ」、Phase 4 物語化 section 新設
- **`#209` sub-5: `walkthrough-script-v1.md` 追加**（commit `cb105bd`）: 5/7-5/14 dogfood log を物語化した 6-7 分 walkthrough script v1 draft、5 act 構成
- **`#270` agent-server: `enqueueForRoom` にルームキュー外側タイムアウト**（5/25、commit `a0a941f`）: room queue dead lock 防止 layer 追加

!!! info "Phase 4 → Phase 5 narrative の質的 jump"
    v0.3.0 は **「組織記憶辞書が動作する道具に転化した release」** として、Phase 4 中盤の `organon paradigm operational realization` を完成 phase に押し上げた釘です。5/16 dogfood の **4 round LLM 訂正 → 1 round 正解** transition が両 agent path（Light + Deep）で N=2 evidence として再現、**「construction → data → middleware → 利用 → 業務」自動 pipeline** が成立しました。詳細な narrative は [organic ontology 概念設計](concepts/organic-ontology.md) を参照。

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
- **tealus-mcp v0.11.0**（5/8、[#260](https://github.com/gamasenninn/tealus/issues/260)）: `send_text_as_file` + `generate_and_send_image` ツール追加（Light v2 機能 parity）。長文 text を file 添付 / DALL-E 3 経由の画像生成 + 投稿の composite action（★ 画像モデルは v0.5.0 [#313](https://github.com/gamasenninn/tealus/issues/313) で `gpt-image-1` に移行済み）
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

    ★ **これは当時の要件です。** v0.5.0 の TypeScript 化（[#330](https://github.com/gamasenninn/tealus/issues/330)）以降、**現在の要件は Node 24** です（[サーバー構築 > 前提条件](admin/server-setup.md) 参照）。

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
