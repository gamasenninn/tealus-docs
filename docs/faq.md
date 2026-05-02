# FAQ（よくある質問）

Tealus の導入・運用でよくある質問をまとめました。

## 導入・運用

### Q: Tealus を動かすのにどんなハードウェアが必要ですか？

NAS（Synology 推奨）または Linux サーバーがあれば動作します。Docker が動く環境であればOKです。

| 項目 | 推奨スペック |
|---|---|
| CPU | 4コア以上 |
| メモリ | 4GB 以上 |
| ストレージ | 50GB 以上 |

詳しくは [サーバー構築](admin/server-setup.md) を参照してください。

### Q: 何人まで使えますか？

50人規模の組織を想定した設計です。10万件のメッセージでの負荷テスト実績があります。

将来的には NAS クラスター構成による水平スケールも計画しています。

### Q: LINE や Slack から移行できますか？

現時点ではデータ移行ツールはありません。Bot API 経由でメッセージのインポートは技術的に可能です。

LINE 連携ブリッジは将来計画として検討中です（[gamasenninn/tealus#160](https://github.com/gamasenninn/tealus/issues/160)）。

### Q: スマホアプリはありますか？

ネイティブアプリはありません。PWA（Progressive Web App）として対応しています。

ブラウザからホーム画面に追加するだけで、アプリのように利用できます。プッシュ通知にも対応しています（iOS / Android / PC）。

詳しくは [インストールと初期設定](guide/setup.md) を参照してください。

### Q: データのバックアップはどうすればよいですか？

以下の2つをバックアップしてください。

- **データベース**: PostgreSQL の `pg_dump` で定期バックアップ
- **メディアファイル**: `media/` ディレクトリを rsync 等でバックアップ

詳しくは [運用・監視](admin/operations.md) を参照してください。

## AI 機能

### Q: AI を使わないことはできますか？

はい。agent-server を起動しなければ AI 機能は無効になります。チャット・通話・トランシーバーは AI なしで問題なく動作します。

### Q: どの AI モデルを使っていますか？

| 用途 | モデル | 提供元 |
|---|---|---|
| Light Agent（日常Q&A） | gpt-5.4-mini | OpenAI |
| Deep Agent（複雑な分析） | Claude | Anthropic |
| Router（振り分け判定） | gpt-5.4-mini | OpenAI |
| 音声合成（TTS） | Aivis Cloud API | Aivis Project |
| 音声文字起こし | gpt-4o-transcribe（v0.2.2〜default、env で whisper-1 戻し可） | OpenAI |

詳しくは [AI アーキテクチャ](dev/ai-architecture.md) を参照してください。

### Q: AI の API 費用はどのくらいですか？

利用頻度によりますが、目安は以下の通りです。

| 項目 | 月額目安 |
|---|---|
| Router | $3〜$13 |
| Light Agent | $80〜$280 |
| Deep Agent（MAXプラン） | $100（固定） |
| TTS 読み上げ（Aivis） | ¥1,980（無制限プラン） |
| 音声文字起こし（gpt-4o-transcribe） | 音声1分あたり約¥0.9（whisper-1 とほぼ同等、`gpt-4o-mini-transcribe` なら半額） |

詳しくは [AIエージェント設定](admin/agent.md) を参照してください。

### Q: AI の回答を音声で聞くにはどうすればよいですか？

プロフィール設定で「AI回答の読み上げ」をONにしてください。メッセージ送信時にトランシーバーが自動接続され、AIの回答が音声で読み上げられます。

詳しくは [TTS 読み上げ](guide/advanced/tts.md) を参照してください。

## セキュリティ

### Q: データは社外に出ますか？

チャットデータ・メディアファイルは完全にオンプレミスで管理されます。

AI 機能を使う場合のみ、メッセージ内容が AI API（OpenAI / Anthropic / Aivis）に送信されます。AI を使わなければ外部への通信は一切発生しません。

### Q: HTTPS に対応していますか？

はい。Cloudflare 経由、またはリバースプロキシ（Nginx / Synology DSM）で SSL/HTTPS に対応します。

詳しくは [サーバー構築](admin/server-setup.md) を参照してください。
