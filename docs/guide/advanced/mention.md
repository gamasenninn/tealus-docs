# メンション

特定のメンバーに通知を送るメンション機能です。

## メンションの使い方

1. 入力欄で `@` を入力
2. メンバー候補のポップアップが表示される
3. メンバーを選択すると `@名前` が挿入される
4. メッセージを入力して送信

!!! note "`@all` / `@here` はありません"
    以前このページには「`@all`（全員）」「`@here`（オンラインの人）」の表がありましたが、**誤りでした**。Tealus にはこれらを特別扱いする処理がなく、書いても通常の文字として扱われ、全員に通知は飛びません。

## メンションの表示

メッセージ内の `@名前` はハイライト表示されます。メンションされた相手には、通常のメッセージよりも優先度の高いプッシュ通知が送信されます。

## メンション履歴

自分がメンションされたメッセージだけをフィルタ表示できます。

## AIアシスタントとの連携

`@アシスタント名` に続けて `/` を入力すると、スラッシュコマンドの候補が表示されます。AIエージェントに対して特定の操作を指示できます。

### `cc-proj` 仮想ユーザー（v0.2.3〜、[#253](https://github.com/gamasenninn/tealus/issues/253)）

メンション picker には、Bot メンバー以外に **`@cc-{project}` 形式の仮想ユーザー**が候補として表示されます。Tealus 上のメッセージから採用者の手元 [Claude Code session（cc-tealus）](../../dev/agents.md#claude-code-session-cc-tealus) を呼び出すための mention で、実体は agent-server の dispatcher が file beacon に追記して Claude Code 側を sub-second で wake する仕組みです。

`agent-server/config/cc-aliases.json` で alias を設定すれば、`@Claude` のような自然な mention 名でも同じ session に routing できます（[#263](https://github.com/gamasenninn/tealus/issues/263)、設定方法は [Agent ガイド](../../dev/agents.md#cc-aliases-json) を参照）。

!!! danger "★ `@cc-…` は「1 行目の行頭に、飾らずに」書いてください"
    `@cc-{project}` が相手を起こすのは、**投稿の 1 行目の先頭に、そのまま書いたとき**です（[#393](https://github.com/gamasenninn/tealus/issues/393) / [#450](https://github.com/gamasenninn/tealus/issues/450)）。次の書き方は**起こしません**:

    | ❌ 起こさない | ✅ 起こす |
    |---|---|
    | `お願いします @cc-docs`（文中） | `@cc-docs お願いします` |
    | `**@cc-docs**`（太字などの飾り） | `@cc-docs` |

    **起こさなかったことは、送った側からは分かりません**（投稿自体は普通に成功して見えます）。

    複数の班へ同時に送るときは、**1 行目にスペース区切りで並べます**（5 班まで、[#386](https://github.com/gamasenninn/tealus/issues/386) / [#387](https://github.com/gamasenninn/tealus/issues/387)）:

    ```
    @cc-docs @cc-support
    v0.11.1 を出しました。…
    ```

    複数の班に届いた便へ返すときは、**先頭の宛先を発信者 1 人だけ**にしてください。受け取った宛先を並べて返すと、班どうしが互いを起こし続けます。
