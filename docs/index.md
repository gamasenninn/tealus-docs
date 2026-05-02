# Tealus ドキュメント

**Tealus** は、オープンソースの社内メッセンジャーです。LINEのような直感的なUIで、画像・動画はサーバーに保存されるため、端末の容量を圧迫しません。

## Tealus の特長

- **直感的なチャットUI** — LINEライクな吹き出しデザイン
- **サーバー保存** — 画像・動画・ファイルはすべてサーバーに保存。端末容量を使わない
- **PWA対応** — スマホ・PCのブラウザからアプリとして利用可能
- **リアルタイム通信** — Socket.IO によるメッセージ即時配信
- **AI連携** — AIエージェントがチャットに参加し、業務を支援
- **音声・ビデオ通話** — WebRTC (mediasoup) によるグループ通話
- **完全自社管理** — オンプレミスで運用可能。データは社外に出ない

## 競合製品との比較

| 機能 | Slack | Teams | Mattermost | Rocket.Chat | **Tealus** |
|------|-------|-------|-----------|------------|-----------|
| テキストチャット | ◎ | ◎ | ○ | ○ | **◎** |
| 音声・ビデオ通話 | ◎ | ◎ | × | △ | **◎** |
| AI エージェント | △ 外部連携 | △ Copilot | △ | △ | **◎ 3層自律型** |
| AI 音声読み上げ（TTS） | × | × | × | × | **◎ 唯一** |
| トランシーバー（PTT） | × | ◎ Walkie Talkie | × | × | **◎** |
| TODO 管理 | × | × | △ | × | **◎ タグベース** |
| 音声文字起こし | × | △ | × | × | **◎ gpt-4o-transcribe + AI整形** |
| 自前ホスティング | × | × | ◎ | ◎ | **◎ NAS 1台** |
| 月額コスト（50人） | ¥50,000〜 | ¥75,000〜 | 無料〜 | 無料〜 | **無料** |
| データ主権 | クラウド | クラウド | 自社 | 自社 | **完全自社** |

※ 2026年4月時点の情報です

## ドキュメントの構成

<div class="grid cards" markdown>

-   :material-account:{ .lg .middle } **ユーザーガイド**

    ---

    基本的な使い方から応用機能まで

    [:octicons-arrow-right-24: はじめに](guide/index.md)

-   :material-shield-account:{ .lg .middle } **管理者ガイド**

    ---

    サーバー構築・ユーザー管理・運用

    [:octicons-arrow-right-24: はじめに](admin/index.md)

-   :material-code-braces:{ .lg .middle } **開発者ガイド**

    ---

    アーキテクチャ・API・拡張開発

    [:octicons-arrow-right-24: はじめに](dev/index.md)

</div>

## クイックリンク

| リソース | リンク |
|---|---|
| ソースコード | [GitHub](https://github.com/gamasenninn/tealus) |
| バグ報告・機能要望 | [Issues](https://github.com/gamasenninn/tealus/issues) |
| ライセンス | MIT License |
