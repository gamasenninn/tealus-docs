# コントリビューション

## 開発環境のセットアップ

[サーバー構築](../admin/server-setup.md)の手順に従って環境を構築してください。

## 開発フロー

1. Issue を作成（機能要望・バグ報告）
2. ブランチを作成
3. 実装・テスト
4. プルリクエストを作成

## コーディング規約

### TypeScript（v0.5.0〜）

v0.5.0（[#330](https://github.com/gamasenninn/tealus/issues/330)）で全レイヤー（server / client / agent-server / rtc-server / dashboard / scripts）が TypeScript 化されました。

- **拡張子**: サーバー系は `.mts`、クライアント系・CLI は `.ts` / `.tsx`
- **ビルド工程はありません**（サーバー系）。`node src/app.mts` のように **`.mts` を直接実行**します（Node の native type stripping）。トランスパイル済みの成果物をコミットする運用ではありません
- **型検査はテストと別工程**です。PR を出す前に、変更したプロジェクトで `npm run typecheck`（= `tsc --noEmit`）を通してください。**テストを実行しても型検査は行われません**
- **Node 24 が必要**です（[サーバー構築 > 前提条件](../admin/server-setup.md)）

!!! tip "テストは `.js` のままでも構いません"
    既存の JS テストは一括変換されておらず、`.test.js` と `.test.mts` が同じ Jest で動く構成になっています。詳細は [テスト](testing.md) を参照してください。

### CLAUDE.md

リポジトリルートの `CLAUDE.md` にAI開発ガイドラインが記載されています。AIツール（Claude Code等）での開発時はこのファイルが自動的に参照されます。Codex CLI 向けには同内容の `AGENTS.md` が置かれています。

### コード品質

- Phase 1 完了時にリファクタリングを実施済み（#51〜#64）
- エラーメッセージは `server/src/constants/errors.mts` に集約
- メンバーシップチェックは共通ミドルウェアを使用
- Socket.IOハンドラは `server/src/socket/handlers/` に機能別分割
- 依存に **GPL / AGPL 系ライセンスを混入させない**（CI の `license-fail-on-gpl` で検査されます）

## セキュリティに関する注意

ドキュメントやコード例を記述する際は、以下のルールを守ってください。

- **シークレット値をサンプルに含めない**: JWT シークレット、DB パスワード、API キー等の例示には `<your_secret>` のような placeholder を使用する
- **弱いパスワードを本番用として記載しない**: `password123` 等を例示する場合は、必ず「localhost 検証用」と明記し `!!! warning` callout を付ける
- **実在のドメインを設定例に使わない**: 設定例には `your-domain.example` 等の placeholder を使用する
- **環境変数のデフォルト値**: 開発用デフォルト値を記載する場合は、本番では変更が必要な旨を明記する

## ライセンス

MIT License
