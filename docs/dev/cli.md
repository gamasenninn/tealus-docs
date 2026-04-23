# CLI ツール

コマンドラインからTealusにメッセージ・画像・音声を送信できます。

## インストール

```bash
npx github:gamasenninn/tealus
```

または、リポジトリ内の `scripts/tealus-cli.js` を直接実行します。

## 設定

`scripts/.env` に認証情報を設定します。

```
TEALUS_SERVER=http://localhost:3000
TEALUS_BOT_ID=BOT001
TEALUS_BOT_PASS=bot_password
```

## コマンド

### テキスト送信

```bash
node tealus-cli.js send "グループ名" --text "Hello!"
```

### 画像送信

```bash
node tealus-cli.js send "グループ名" --image /path/to/image.jpg
```

### 音声送信

```bash
node tealus-cli.js send "グループ名" --voice /path/to/audio.wav
```

### ダイレクトメッセージ

`@ユーザー名` を宛先に指定するとDMルームが自動作成されます。

```bash
node tealus-cli.js send "@山田太郎" --text "こんにちは"
```

## watchモード（ディレクトリ監視）

ディレクトリを監視し、新しい音声ファイルが追加されると自動でTealusに送信します。

```bash
node tealus-cli.js send "グループ名" --watch /path/to/directory
```

### オプション

| オプション | 説明 |
|---|---|
| `--ext .wav,.mp4,.mp3` | 対象ファイル拡張子（カンマ区切り） |
| `--catch-up` | 未送信ファイルの自動送信 |

!!! info "ファイル書き込み完了検知"
    watchモードはファイルサイズが2回連続で同一になるまで待機してから送信します。書き込み途中のファイルが送信されることはありません。

## VOX音声連携

トランシーバーのVOX音声出力ディレクトリをwatchモードで監視することで、トランシーバーの音声を自動的にTealusに投稿できます。

```bash
# トランシーバー音声をTealusに自動投稿
node tealus-cli.js send "現場連絡" --watch /path/to/vox-output --ext .wav
```
