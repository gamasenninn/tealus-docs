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

LINE とは**連携ブリッジがあります**（LINE のグループと Tealus の部屋をつなぐ。過去の履歴の移行ではありません）。詳しくは [LINE Bridge 連携](guide/integration/line-bridge.md) を参照してください。

### Q: スマホアプリはありますか？

ネイティブアプリはありません。PWA（Progressive Web App）として対応しています。

ブラウザからホーム画面に追加するだけで、アプリのように利用できます。プッシュ通知にも対応しています（iOS / Android / PC）。

詳しくは [インストールと初期設定](guide/setup.md) を参照してください。

### Q: データのバックアップはどうすればよいですか？

以下の2つをバックアップしてください。

- **データベース**: PostgreSQL の `pg_dump` で定期バックアップ
- **メディアファイル**: `media/` ディレクトリを rsync 等でバックアップ
- **鍵と設定**: `.env`（`JWT_SECRET` や VAPID 鍵）。これが無いと、戻しても全員が再ログインになりプッシュが無効になります

!!! danger "★ Windows の PowerShell で `pg_dump ... > backup.sql` を使わないでください"
    `>` が文字コードを変えて**ダンプが壊れ、復元しようとするまで気づけません**（[#566](https://github.com/gamasenninn/tealus/issues/566)）。コンテナの中でファイルに書いてから `docker cp` で取り出してください。

詳しい手順は [運用・監視](admin/operations.md) を参照してください。

## AI 機能

### Q: AI を使わないことはできますか？

はい。agent-server を起動しなければ AI 機能は無効になります。チャット・通話・トランシーバーは AI なしで問題なく動作します。

### Q: どの AI モデルを使っていますか？

| 用途 | モデル | 提供元 |
|---|---|---|
| Light Agent v2（既定） | codex SDK。モデルは `AGENT_LIGHT_MODEL`（`.env.example` の値は `gpt-5.6-sol`。★ 未設定時のコード上の既定 `gpt-5.4-mini` は ChatGPT アカウントでは使えないので、**必ず設定してください**） | OpenAI |
| Light Agent v1 | gpt-5.4-mini（API キー） | OpenAI |
| Deep Agent | Claude Code CLI（既定）／ Codex CLI（`gpt-5.6-sol`） | Anthropic ／ OpenAI |
| Router（振り分け判定） | gpt-5.4-mini（API キー） | OpenAI |
| 音声合成（TTS） | Aivis Cloud ／ OpenAI ／ Gemini ／ ブラウザ内蔵（`TTS_PROVIDER` で選択） | 各社 |
| 音声文字起こし | **gpt-4o-mini-transcribe**（既定、`WHISPER_MODEL` で変更可）／自ホスト／ Gemini（`STT_BACKEND`） | OpenAI ほか |

!!! warning "ChatGPT アカウント（サブスクリプション）で Codex を使う場合"
    **mini 系のモデルは使えません**（400 になります）。`gpt-5.5` は 2026-10-14 に外され、既定は `gpt-5.6-sol` です。**モデル名を替えるだけでなく `cd agent-server && npm install` が必要**です（codex-sdk 0.162.1）。詳しくは [AIエージェント設定](admin/agent.md) を参照してください。

詳しくは [AI アーキテクチャ](dev/ai-architecture.md) を参照してください。

### Q: AI の API 費用はどのくらいですか？

利用頻度によりますが、目安は以下の通りです。

| 項目 | 月額目安 |
|---|---|
| Router | $3〜$13 |
| Light Agent | $80〜$280 |
| Deep Agent（MAXプラン） | $100（固定） |
| TTS 読み上げ（Aivis） | ¥1,980（無制限プラン） |
| 音声文字起こし（既定 `gpt-4o-mini-transcribe`） | `gpt-4o-transcribe` の約半額（既定を替えた理由は幻覚の抑制で、料金ではありません。[#284](https://github.com/gamasenninn/tealus/issues/284)） |

詳しくは [AIエージェント設定](admin/agent.md) を参照してください。

### Q: AI の回答を音声で聞くにはどうすればよいですか？

プロフィール設定で「AI回答の読み上げ」をONにしてください。AI の回答が届くと、その端末で読み上げられます（トランシーバーには接続しません）。

詳しくは [TTS 読み上げ](guide/advanced/tts.md) を参照してください。

## セキュリティ

### Q: データは社外に出ますか？

チャットデータ・メディアファイルは完全にオンプレミスで管理されます。

AI 機能を使う場合、メッセージ内容が AI API（OpenAI / Anthropic / Aivis / Google Gemini）に送信されます。

★ **AI を使わなくても、次の外部通信はあります**（以前このページは「外部への通信は一切発生しない」と書いていましたが、誤りでした）:

| 通信 | 送られるもの | 止め方 |
|---|---|---|
| リンクプレビュー | 投稿に含まれる URL へ、server がページを取りに行く（社内のアドレスへは取りに行かない） | — |
| プッシュ通知 | 通知が Apple / Google / Mozilla の配信サービスを経由する（中身は端末の鍵で暗号化され、配信サービスは読めない。宛先と時刻・大きさは分かる） | VAPID 鍵を設定しない |
| 文字起こし・読み上げ | 音声・文が OpenAI / Google（`STT_BACKEND=gemini`・`TTS_PROVIDER=gemini` のとき）へ | 読み上げは `TTS_PROVIDER=browser`（または `none`）。文字起こしは `STT_BACKEND=local` に加えて `STT_LOCAL_STRICT=1`（失敗時に OpenAI へ切り替えない）。★ ただし**文字起こしの整形（AI 整形）は経路に関係なく OpenAI に文を送ります**。完全に止めるには音声の文字起こし自体を使わない運用になります |

### Q: HTTPS に対応していますか？

はい。Cloudflare 経由、またはリバースプロキシ（Nginx / Synology DSM）で SSL/HTTPS に対応します。

詳しくは [サーバー構築](admin/server-setup.md) を参照してください。
