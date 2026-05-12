# Claude Code 連携

Tealus は **Claude Code (Anthropic 製の CLI)** との双方向統合を OSS 公開直後 (v0.2.0) から備えています。本ページは **採用者向けの概念紹介**で、技術的な step-by-step 手順は本体 repo の [`setup-cc-tealus-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-cc-tealus-bridge.md) を参照してください。

## 全体像 — 2 方向の統合

```
[Claude Code]                   [Tealus]
     │                              │
     │  ────── Outbound (1) ─────►  │   tealus-mcp 経由
     │  Tealus に投稿 / 検索 / etc.   │   AI が能動的に Tealus を扱う
     │                              │
     │  ◄────── Inbound (2) ─────   │   cc-tealus bridge
     │  @cc-{project} / @Claude     │   Tealus から Claude Code を起こす
     │                              │
```

### Outbound — Claude Code → Tealus（`tealus-mcp`）

Claude Code（または Cursor 等の MCP クライアント）から **Tealus にメッセージ投稿 / 履歴検索 / 画像生成 + 投稿** を実行できる経路です。AI が能動的なメンバーとして Tealus を扱えるようになります。

- 詳細仕様: [MCP Server](../../dev/mcp.md)
- repo: [`gamasenninn/tealus-mcp`](https://github.com/gamasenninn/tealus-mcp)
- 配信: `npx -y github:gamasenninn/tealus-mcp`（npm publish ではなく GitHub 直接 install）

### Inbound — Tealus → Claude Code（cc-tealus bridge）

Tealus 上で `@cc-{project}` / `@Claude` mention を投げると、採用者の手元の Claude Code session が **sub-second で wake up** して応答します（file beacon + Monitor 経路）。

- 詳細仕様: [Agent ガイド > Claude Code session（cc-tealus）](../../dev/agents.md#claude-code-session-cc-tealus)
- alias 設定: [`cc-aliases.json`](../../dev/agents.md#cc-aliases-json)（`@Claude` 等の自然な mention 名を routing）

## Transport の選び分け — stdio default / HTTP optional

tealus-mcp は **2 つの transport** を提供します（v0.12.0+、[tealus #264](https://github.com/gamasenninn/tealus/issues/264) Phase 1 alpha）。

| transport | 用途 | 構成 | 認証 |
|-----------|------|------|------|
| **stdio**（default、v0.2.0+） | **同一マシン** で Tealus と Claude Code を動かす一般的な構成 | Claude Code が `npx` で tealus-mcp を child process spawn | env (`TEALUS_USER_ID` / `TEALUS_PASSWORD`) |
| **HTTP**（opt-in、v0.12.0+） | **cross-machine 構成** — Tealus サーバと Claude Code が別マシン | Tealus 本体側で tealus-mcp を HTTP server として常駐、Claude Code は URL で接続 | JWT (`JWT_SECRET` を 3 process で共有) |

### stdio の典型ユースケース

```json
// ~/.claude.json (project 単位の mcpServers block)
{
  "mcpServers": {
    "tealus": {
      "command": "npx",
      "args": ["-y", "github:gamasenninn/tealus-mcp"],
      "env": {
        "TEALUS_API_URL": "http://localhost:3000",
        "TEALUS_USER_ID": "<bot_user_id>",
        "TEALUS_PASSWORD": "<bot_password>"
      }
    }
  }
}
```

- Tealus 本体と Claude Code が**同じマシン**にある（個人 dogfood、自宅サーバ、開発機）
- zero-config に近い、env 3 つで起動

### HTTP の典型ユースケース

```
[Claude Code (マシン A)]              [Tealus サーバ (マシン B)]
  ~/.claude.json                          port 3000 (Tealus 本体)
  mcpServers:                             ┌──────────────────┐
    tealus:                               │ /mcp proxy       │
      url: https://<host>/mcp             │   ↓              │
      headers:                            │ port 3200        │
        Authorization: Bearer <JWT> ────► │ tealus-mcp HTTP  │
                                          └──────────────────┘
```

- Tealus サーバが**社内サーバ / クラウド**に居て、Claude Code は採用者のローカル PC
- HTTPS / reverse proxy（nginx 等）で TLS 終端、JWT で auth

!!! info "Phase 1 alpha の限界"
    Phase 1 alpha は **HTTP request/response（tools 呼び出し）のみ**。Inbound 通知（Tealus → Claude Code の wake-up）は **stdio + file beacon 経路に依存**します。cross-machine 構成では、wake-up 用に **stdio + file beacon は同一マシンが必要**で、HTTP は tools 呼び出し補完として使う構図です。Phase 2 で SSE event broker（server-push wake-up）が乗ると HTTP 単独で完結する予定です。

## どちらを使うべきか

| 状況 | 推奨 transport |
|---|---|
| 個人 dogfood、自宅マシン | **stdio** (zero-config) |
| 開発機 + ローカル Tealus 本体 | **stdio** |
| 採用者のオフィスマシン、社内 Tealus サーバ | **HTTP** (cross-machine 必須) |
| 比較 / 移行期 | **両方並列** に登録して動作確認、HTTP に切り替えた後に stdio を削除 |

## セットアップ手順

技術的な step-by-step は本体 repo の **[`setup-cc-tealus-bridge.md`](https://github.com/gamasenninn/tealus/blob/main/docs/setup-cc-tealus-bridge.md)** に集約しています。

- **Outbound（tealus-mcp、stdio）**: ステップ 1A〜5
- **Outbound（tealus-mcp、HTTP）**: ステップ 5A（cross-machine 構成）
- **Inbound（cc-tealus bridge）**: ステップ 1B〜5B（Tealus → Claude Code 経路）
- **統合動作確認**: ステップ 6（双方向が 1 cycle で繋がる流れ）

## 関連

- [MCP Server](../../dev/mcp.md): 15 ツールの詳細仕様 + HTTP transport セクション
- [Agent ガイド](../../dev/agents.md): cc-tealus bridge / Light v1 v2 / Deep / cc-aliases.json
- [リリースノート](../../releases.md): tealus-mcp v0.12.x group（HTTP transport 4 release）
- [tealus #264](https://github.com/gamasenninn/tealus/issues/264) HTTP transport Phase 1 alpha
- [tealus #213](https://github.com/gamasenninn/tealus/issues/213) cc-tealus Phase A
