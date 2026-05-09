# メンション

特定のメンバーに通知を送るメンション機能です。

## メンションの使い方

1. 入力欄で `@` を入力
2. メンバー候補のポップアップが表示される
3. メンバーを選択すると `@名前` が挿入される
4. メッセージを入力して送信

## 全員メンション

| メンション | 対象 |
|---|---|
| `@all` | ルームの全メンバーに通知 |
| `@here` | 現在オンラインのメンバーに通知 |

## メンションの表示

メッセージ内の `@名前` はハイライト表示されます。メンションされた相手には、通常のメッセージよりも優先度の高いプッシュ通知が送信されます。

## メンション履歴

自分がメンションされたメッセージだけをフィルタ表示できます。

## AIアシスタントとの連携

`@アシスタント名` に続けて `/` を入力すると、スラッシュコマンドの候補が表示されます。AIエージェントに対して特定の操作を指示できます。

### `cc-proj` 仮想ユーザー（`[Unreleased]`、[#253](https://github.com/gamasenninn/tealus/issues/253)）

メンション picker には、Bot メンバー以外に **`@cc-{project}` 形式の仮想ユーザー**が候補として表示されます。Tealus 上のメッセージから採用者の手元 [Claude Code session（cc-tealus）](../../dev/agents.md#claude-code-session-cc-tealus) を呼び出すための mention で、実体は agent-server の dispatcher が file beacon に追記して Claude Code 側を sub-second で wake する仕組みです。

`agent-server/config/cc-aliases.json` で alias を設定すれば、`@Claude` のような自然な mention 名でも同じ session に routing できます（[#263](https://github.com/gamasenninn/tealus/issues/263)、設定方法は [Agent ガイド](../../dev/agents.md#cc-aliases-json) を参照）。
