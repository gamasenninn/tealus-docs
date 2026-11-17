# 運用・監視

## ログ管理

### サーバーログ

Tealus Server は `winston` + `winston-daily-rotate-file` でログを出力します。

- **出力先**: `server/logs/tealus-server-YYYY-MM-DD.log`
- **保持期間**: 14日間（自動ローテーション）
- **ログレベル**: `LOG_LEVEL` 環境変数で制御
    - 開発: `debug`（Socket.IO接続/切断、メッセージ送信、認証等の詳細ログ）
    - 本番: `info`（重要なイベントのみ）

v0.12.0 から、次の行が増えました。warn を数えて見張っている場合は数え直してください。

| 行 | レベル | 中身 |
|---|---|---|
| `[http-denied]` | warn | 断った要求（状態・方法・パス・利用者・ip・ua・理由 `reason=`） |
| `[change]` | info | 成功した変更（GET 以外）。誰が・パス・本文の欄の名前（値は残さない）。AI が頼まれて編集・削除したときは頼んだ人も |
| `[http-flood]` | warn | 同じ相手が同じ口を 1 分に 60 回を超えて叩いた（10 分に 1 回まで。止めはしない） |
| `[http]` | warn | 4xx の応答（以前は `[error]` でした） |

agent-server にも同じ行があります。

### ログビューア

システムダッシュボード（`/system`）の **モニタリング → サーバーログ** タブで確認できます。

## エージェント監視

### 稼働モニタリング

システムダッシュボード（`/system`）の「モニタリング」で以下を確認できます。

| タブ | 内容 |
|---|---|
| リアルタイム | エージェントの状態と処理状況をリアルタイム表示 |
| 応答ログ | 質問と応答の詳細記録（ルームで絞り込み可） |
| サーバーログ | server のログ |

（以前このページにあった「統計」タブは、画面にはありません。統計は下の API で取れます。）

### API

```
GET /api/admin/agent-stats   # 統計情報
GET /api/admin/agent-logs    # 応答ログ
```

## Webhook管理

管理ダッシュボード（`/admin`）の「Webhook」タブから管理します。

| 操作 | 説明 |
|---|---|
| 登録 | Webhook URLとイベント種別を設定 |
| テスト送信 | テスト用のWebhookペイロードを送信 |
| 有効/無効切替 | Webhookの一時停止・再開 |
| 削除 | Webhook設定の削除 |

### セキュリティ

Webhookペイロードは `X-Tealus-Signature: sha256=HMAC(secret, body)` ヘッダで署名されます。受信側で署名を検証することで改ざんを防止できます。

!!! danger "★ agent-server を使う場合、secret は両側に同じ値が要ります"
    [#478](https://github.com/gamasenninn/tealus/issues/478) から、agent-server の受け口も署名を検証します。**本体の Webhook 登録に入れた secret と、`agent-server/.env` の `WEBHOOK_SECRET` に同じ値**を設定してください。

    **agent-server 側に `WEBHOOK_SECRET` があり、本体側の secret が無い・違う**と、**401 で弾かれ、エージェントが黙って起きません**（投稿は普通に成功して見えるため、送った側からは気づけません）。

    逆に **agent-server 側が未設定**だと、署名を検査せずに受け付けます（起動時に警告が出るだけで動きます）。動くからといって安全ではないので、両側に設定してください。`agent-server` の `npm run doctor` でも未設定を検出できます。

### リトライ

配信は**最大 3 回試行**します（失敗時に 5 秒後・15 秒後の 2 回再試行）。**4xx は再試行しません**（受け側の設定誤りは繰り返しても直らないため）。

## 設定の点検（`npm run doctor`）

agent-server には、設定の抜けや食い違いを点検する口があります（[#438](https://github.com/gamasenninn/tealus/issues/438) [#442](https://github.com/gamasenninn/tealus/issues/442)）。

```bash
cd agent-server
npm run doctor             # 既定: 外部の API は叩かない
npm run doctor -- --probe  # 外部（OpenAI 等）への疎通も確かめる
```

起動時の 1 行と同じ判定を、手で確かめたいときに使います。必須の環境変数・`WEBHOOK_SECRET`・資格情報・Codex のモデル・未適用の migration などを見ます。v0.12.0 からは、メディアの総量・ディスクの空き・常駐のメモリと、「DB が指すのにファイルが無い添付」の数も出します。値そのものは表示しません（指紋のみ）。叩かなかった経路は `not-probed` と出るので、「確かめて全部だめ」と区別できます。

## ヘルスチェック

```
GET /api/health
```

監視ツールからのヘルスチェック用エンドポイントです。

## バックアップ

### データベース

PostgreSQLの標準的なバックアップ手法を使用します。

```bash
# 開発用（docker-compose.yml、コンテナ tealus_postgres）
docker-compose exec -T postgres pg_dump -U tealus tealus > backup-$(date +%Y%m%d).sql

# フル構成（docker-compose.full.yml、コンテナ tealus_postgres_prod）
docker compose -f docker-compose.full.yml exec -T postgres pg_dump -U tealus tealus > backup-$(date +%Y%m%d).sql
```

★ コンテナ名は構成で違います。**開発用は `tealus_postgres`、フル構成は `tealus_postgres_prod`** です（下の PowerShell の例もこれに合わせて置き換えてください）。`-f docker-compose.full.yml` を付け忘れると、フル構成の DB には届きません。

!!! danger "★ Windows の PowerShell 5.1 では上の `>` を使わないでください"
    PowerShell の `>` が**文字コードを変えてしまい、ダンプが壊れます**（[#566](https://github.com/gamasenninn/tealus/issues/566)）。**壊れたことは復元しようとするまで分かりません。**

    コンテナの中でファイルに書いてから取り出してください（custom 形式）:

    ```powershell
    # フル構成なら tealus_postgres_prod に置き換える
    docker exec tealus_postgres pg_dump -U tealus -Fc -f /tmp/backup.dump tealus
    docker cp tealus_postgres:/tmp/backup.dump .\backup.dump
    ```

!!! warning "DB だけでは戻せません"
    復元には次の 3 つが揃っている必要があります。1 つでも欠けると、戻したつもりで欠けた状態になります。

    | 対象 | 内容 | 欠けるとどうなるか |
    |---|---|---|
    | DB のダンプ | `pg_dump` の出力 | 投稿・利用者・部屋がすべて失われる |
    | メディア | `MEDIA_ROOT` の中身（画像・動画・音声・ファイル・サムネイル） | 投稿は残るが、添付が開けない |
    | 鍵と設定 | `.env`（`JWT_SECRET` / VAPID 鍵など） | 全員が再ログイン、プッシュの購読が無効になる |
    | AI の設定 | agent-server の `.env`・`config/`・部屋の設定（`_room-configs/`） | 応答モード・部屋の指示・道具の設定が消える |

    Windows では `scripts/backup/tealus-backup.ps1 -Dest <UNC>` で、DB・メディア・鍵と設定・記録・AI の設定をまとめて写せます（`-Parts` で一部だけ）。DB の控えの横に件数の控え（`.counts.json`）を置くので、戻す試しで比べられます。

    復元の具体的な手順は、本体リポジトリの [upgrade-guide.md](https://github.com/gamasenninn/tealus/blob/main/docs/upgrade-guide.md) の「壊れたときの復元」にあります。

## Windows で自動起動する {#autostart}

`scripts/startup/tealus-autostart.ps1` は、ログオン時に本体・agent-server・rtc-server を上げます（v0.12.0）。タスクスケジューラへの登録は各自で行います。`-DryRun` で何を起動するか試せます。環境ごとに違う起動は、リポジトリの外の `~\.tealus\startup\extras.ps1` に書きます。

### メディアファイル

`media/` ディレクトリにアップロードされたファイルが保存されています。定期的にバックアップしてください。
