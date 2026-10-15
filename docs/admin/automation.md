# ルームトリガーと見張り

決まった仕事を自動で投げる「**ルームトリガー**」と、来るはずのものが来ないときに知らせる「**見張り**」です。どちらも server に入っていて、**設定ファイル（JSON）を置くだけ**で動きます。

| | ルームトリガー | 見張り |
|---|---|---|
| すること | 条件が満たされたら、決まった部屋に決まった文を投稿する | 止まったものがあれば、業務メモの部屋に 1 回だけ知らせる |
| 設定ファイル | `server/config/room-triggers.json` | `server/config/watchdogs.json` |
| 見本 | `room-triggers.example.json` | `watchdogs.example.json` |
| 判定の間隔 | 10 秒ごと | 5 分ごと（起動 2 分後に初回） |
| 設計書（本体） | [06_ルームトリガー設計.md](https://github.com/gamasenninn/tealus/blob/main/docs/06_ルームトリガー設計.md) | [09_見張り設計.md](https://github.com/gamasenninn/tealus/blob/main/docs/09_見張り設計.md) |

共通の作法:

- **設定は判定のたびに読み直します。再起動は要りません**
- ファイルが無いのは正常（未設定）。JSON が壊れていても**本体の起動は止めず**、壊れた行を落として**必ず警告を記録に出します**
- 止めたいときは行を消さずに `"enabled": false`

## ルームトリガー

**「条件が満たされたら、決まったメッセージを、決まった部屋に、決まった名義で投稿する」** だけの機能です（[#382](https://github.com/gamasenninn/tealus/issues/382)）。投稿の先は今の仕組みにそのまま乗るので、本文の 1 行目に `@アシスタント` や `@cc-…` を書けば、エージェントがいつもどおり起きます。

```json
[
  {
    "id": "chourei-gijiroku",
    "room_id": "<部屋の ID>",
    "room": "朝礼",
    "types": ["video"],
    "when": "immediate",
    "message": "@アシスタント /light この動画から議事録を作成して",
    "as_user_id": "<名義にする利用者の ID>",
    "enabled": true,
    "quiet_minutes": 3
  }
]
```

| `when` | 撃つ条件 | 追加の項目 |
|---|---|---|
| `immediate` | `types` の投稿が来て、まだ撃っていない | `quiet_minutes`（最後の投稿から静かになるまで待つ。既定 3 分。続けて 2 本上がった日に 2 回撃たないため [#385](https://github.com/gamasenninn/tealus/issues/385)） |
| `every` | 前回撃ってから `types` の投稿があった | `interval_minutes` |
| `schedule` | 指定の時刻を過ぎ、その日まだ撃っていない | `at`（日本時間、例 `"08:00"`） |

- **自動投稿には 2 行目に印が付きます**（`— 自動投稿 (room-triggers: <id>)`）。人が打ったのか仕組みが撃ったのかが、後から分かります。この印で「前回いつ撃ったか」を部屋から引くので、**`id` を変えると履歴が切れます**
- `types` に **`text` は原則使えません**（エージェントの返事を材料にして撃ち続けるループを防ぐため）。送り手を `from_user_ids` で名指ししたトリガーだけ例外です（[#433](https://github.com/gamasenninn/tealus/issues/433)）
- 本文の内容による条件（「〜を含んだら」）はありません。種別と時刻だけです
- `room` は人が読むための名前です。照合は `room_id` で行います

!!! danger "★ メンション先は検証されません"
    `@cc-tealus-app` と `@cc-tealus-apps` のような 1 文字違いでも、**投稿は成功し、警告も出ず、誰も起きません**。設定を書くときは、**その部屋で実際に使われているメンションをコピー**してください。

## 見張り

本番には「毎日・毎時に起きるはずのもの」があります。それが止まったとき、**業務メモの部屋に知らせます**（[#595](https://github.com/gamasenninn/tealus/issues/595)）。知らせるのは**変わったときだけ**です（正常 → 異常で ⚠ を 1 回、異常 → 正常で ✅ を 1 回）。

```json
{
  "notify": { "room_id": "<業務メモの部屋の ID>", "room": "業務メモ", "as_user_id": "<名義の ID>" },
  "watches": [
    {
      "id": "radio-sender",
      "name": "無線の送り込み (tealus-cli)",
      "kind": "heartbeat",
      "heartbeat": "radio-sender",
      "from_user_id": "<鼓動を送る利用者の ID>",
      "max_silence_minutes": 10,
      "max_send_lag_minutes": 15,
      "enabled": true
    }
  ]
}
```

| `kind` | 何を見るか | 異常とみなすとき |
|---|---|---|
| `heartbeat` | 部品が `POST /api/bot/heartbeat` で送る**鼓動** | `max_silence_minutes` を超えて来ない／一度も来ていない／最新のファイルは新しいのに送っていない（`max_send_lag_minutes`） |
| `room_quiet` | 部屋の、指定した送り手（`from_user_ids`）の最新の投稿 | 稼働時間（`hours`・`days`）の中で `max_quiet_minutes` を超えて投稿が無い |
| `backup_log` | Tealus のバックアップの記録（`dir` の `backup-YYYYMMDD.log`） | 最後の記録が成功で終わっていない／26 時間より古い／置き場が読めない |

鼓動は CLI の監視モードから送れます:

```bash
node scripts/tealus-cli.ts send "<部屋>" --voice --watch <dir> --heartbeat radio-sender
```

1 分ごとに、監視しているフォルダの最新のファイルの時刻と、最後に送った時刻を送ります。鼓動は部屋には流れません。

!!! note "Tealus が見るのは自分の境界まで"
    見張りが見るのは、Tealus の部品・Tealus の部屋・Tealus のバックアップだけです。外部の機器の中（録音・文字起こしの装置など）が動いているかは、その機器の側から Tealus へ投稿して知らせる形にします。
