# 運用・監視

## ログ管理

### サーバーログ

Tealus Server は `winston` + `winston-daily-rotate-file` でログを出力します。

- **出力先**: `server/logs/tealus-server-YYYY-MM-DD.log`
- **保持期間**: 14日間（自動ローテーション）
- **ログレベル**: `LOG_LEVEL` 環境変数で制御
    - 開発: `debug`（Socket.IO接続/切断、メッセージ送信、認証等の詳細ログ）
    - 本番: `info`（重要なイベントのみ）

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

    片方だけだと **401 で弾かれ、エージェントが黙って起きません**（投稿は普通に成功して見えるため、送った側からは気づけません）。`agent-server` の `npm run doctor` でも未設定を検出できます。

### リトライ

配信は**最大 3 回試行**します（失敗時に 5 秒後・15 秒後の 2 回再試行）。**4xx は再試行しません**（受け側の設定誤りは繰り返しても直らないため）。

## 設定の点検（`npm run doctor`）

agent-server には、設定の抜けや食い違いを点検する口があります（[#438](https://github.com/gamasenninn/tealus/issues/438) [#442](https://github.com/gamasenninn/tealus/issues/442)）。

```bash
cd agent-server
npm run doctor             # 既定: 外部の API は叩かない
npm run doctor -- --probe  # 外部（OpenAI 等）への疎通も確かめる
```

起動時の 1 行と同じ判定を、手で確かめたいときに使います。必須の環境変数・`WEBHOOK_SECRET`・資格情報・Codex のモデル・未適用の migration などを見ます。値そのものは表示しません（指紋のみ）。叩かなかった経路は `not-probed` と出るので、「確かめて全部だめ」と区別できます。

## ヘルスチェック

```
GET /api/health
```

監視ツールからのヘルスチェック用エンドポイントです。

## バックアップ

### データベース

PostgreSQLの標準的なバックアップ手法を使用します。

```bash
docker-compose exec -T postgres pg_dump -U tealus tealus > backup-$(date +%Y%m%d).sql
```

!!! danger "★ Windows の PowerShell 5.1 では上の `>` を使わないでください"
    PowerShell の `>` が**文字コードを変えてしまい、ダンプが壊れます**（[#566](https://github.com/gamasenninn/tealus/issues/566)）。**壊れたことは復元しようとするまで分かりません。**

    コンテナの中でファイルに書いてから取り出してください（custom 形式）:

    ```powershell
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

    復元の具体的な手順は、本体リポジトリの [upgrade-guide.md](https://github.com/gamasenninn/tealus/blob/main/docs/upgrade-guide.md) の「壊れたときの復元」にあります。

### メディアファイル

`media/` ディレクトリにアップロードされたファイルが保存されています。定期的にバックアップしてください。
