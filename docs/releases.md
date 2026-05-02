# リリースノート

## v0.2.2 — cc-tealus + transcription quality jump（2026年5月2日）

業務メモから 5/2 朝に届いた現場声 3 件を 6 時間サイクルで Issue 起票 → 実装 → close まで完遂したリリース。Tealus の self-improving 哲学を release process そのもので体現。

- **cc-tealus リアルタイム連携 Phase A**（[#213](https://github.com/gamasenninn/tealus/issues/213)）: agent-server が `@cc-{project}` mention を検知して file beacon 経由で Claude Code session を sub-second wake-up。agent-server を「**AI 班 dispatcher**」へ進化させる構造判断の入口
- **voice 再文字起こし機能**（[#216](https://github.com/gamasenninn/tealus/issues/216)）: 文字起こしが失敗 / 不適切な voice メッセージを手動 retry 可能に。新 endpoint `POST /api/messages/:id/transcription/retranscribe`
- **gpt-4o-transcribe default 採用**（[#217](https://github.com/gamasenninn/tealus/issues/217)）: whisper-1 の hallucination（TV 字幕由来 noise 等）が大幅軽減。env `WHISPER_MODEL` で旧モデル戻し可
- **tealus-cli credentials defensive**（[#212](https://github.com/gamasenninn/tealus/issues/212)）: watch モードで JWT 再取得時の再発バグに対する起動時 freeze 防御層

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
