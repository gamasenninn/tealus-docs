# CLI ツール

コマンドラインからTealusにメッセージ・画像・音声を送信できます。

## インストール

```bash
npx github:gamasenninn/tealus
```

または、リポジトリ内の `scripts/tealus-cli.ts` を直接実行します（v0.5.0 の TypeScript 化で `.js` → `.ts` になりました。ビルドは不要で、Node 24 がそのまま実行します）。

## 設定

`scripts/.env` に認証情報を設定します。

```
TEALUS_SERVER=http://localhost:3000
TEALUS_BOT_ID=<your_bot_id>
TEALUS_BOT_PASS=<your_bot_password>
```

!!! warning
    `<your_bot_id>` と `<your_bot_password>` は実際のBotユーザーの認証情報に置き換えてください。

## コマンド

### テキスト送信

```bash
node scripts/tealus-cli.ts send "グループ名" --text "Hello!"
```

### 画像送信

```bash
node scripts/tealus-cli.ts send "グループ名" --image /path/to/image.jpg
```

### 音声送信

```bash
node scripts/tealus-cli.ts send "グループ名" --voice /path/to/audio.wav
```

### 未読の確認・部屋の一覧

```bash
node scripts/tealus-cli.ts check                 # 全部屋の未読
node scripts/tealus-cli.ts check "グループ名"     # その部屋だけ
node scripts/tealus-cli.ts check --mark-read     # 表示した分を既読にする
node scripts/tealus-cli.ts check --json          # JSON で出す
node scripts/tealus-cli.ts rooms                 # 参加している部屋の一覧
```

### ダイレクトメッセージ

`@ユーザー名` を宛先に指定するとDMルームが自動作成されます。

```bash
node scripts/tealus-cli.ts send "@山田太郎" --text "こんにちは"
```

## watchモード（ディレクトリ監視）

ディレクトリを監視し、新しい音声ファイルが追加されると自動でTealusに送信します。

```bash
node scripts/tealus-cli.ts send "グループ名" --voice --watch /path/to/directory
```

!!! warning "`--voice` を省かないでください"
    監視モードは **`--voice` と組み合わせたときだけ**有効になります。以前このページの例は `--voice` が抜けていて、そのとおりに打つと監視モードに入りません。

### オプション

| オプション | 説明 |
|---|---|
| `--ext .wav,.mp4,.mp3` | 対象ファイル拡張子（カンマ区切り、既定 `.wav`） |
| `--catch-up` | 起動時に未送信ファイルを自動送信 |
| `--heartbeat <名前>` | 監視中、**1 分ごとに本体へ鼓動を送る**（[#595](https://github.com/gamasenninn/tealus/issues/595)）。本体の[見張り](../admin/automation.md)が、止まったこと・送り込みの詰まりに気づけます。名前は英数字と `. _ : -` だけ |

### グローバルオプション

| オプション | 説明 |
|---|---|
| `--env <file>` | 読み込む env ファイルを指定（例: `--env .env.trx`）。1 台で複数の bot を使い分けるとき |
| `--bot-id <ID>` / `--bot-pass <PASS>` | bot の認証情報。env ファイルより優先 |

設定が無いと「`--bot-id / --bot-pass を指定するか、scripts/.env に TEALUS_BOT_ID と TEALUS_BOT_PASS を設定してください`」で止まります。

!!! info "ファイル書き込み完了検知"
    watchモードはファイルサイズが2回連続で同一になるまで待機してから送信します。書き込み途中のファイルが送信されることはありません。

!!! note "v0.2.2〜の credentials 保護"
    watch モードは長時間動作するため、JWT 期限切れ時に再認証が発生します。v0.2.2（[#212](https://github.com/gamasenninn/tealus/issues/212)）から起動時に `Object.freeze` で認証情報を固定 capture することで、runtime 中の credentials 漏洩から保護されています。動作上の変更はありません（再発性 bug への防御層追加）。

## VOX音声連携

トランシーバーのVOX音声出力ディレクトリをwatchモードで監視することで、トランシーバーの音声を自動的にTealusに投稿できます。

```bash
# トランシーバー音声をTealusに自動投稿
node scripts/tealus-cli.ts send "現場連絡" --voice --watch /path/to/vox-output --ext .wav
```
