# テスト

Tealus は v0.5.0（[#330](https://github.com/gamasenninn/tealus/issues/330)）で全レイヤーが TypeScript 化され、**テストの実行と型検査が別々の工程**になりました。

- **テスト** — Jest / Vitest で実行（下記）
- **型検査** — 各プロジェクトの `npm run typecheck`（= `tsc --noEmit`）。テスト実行時には型検査は行われません

!!! info "前提"
    **Node 24** が必要です（[サーバー構築 > 前提条件](../admin/server-setup.md)）。

## サーバーテスト（Jest）

```bash
cd server

# テスト用DB起動済みであること（docker-compose up -d）
npm test           # 全テスト実行
npm run test:watch # ウォッチモード
npm run typecheck  # 型検査（tsc --noEmit）
```

テスト用DBは Docker Compose で Port 5433 に起動されます。テストは `--runInBand`（直列実行）で走ります。

## クライアントテスト（Vitest）

```bash
cd client
npm test           # 全テスト実行（vitest run）
npm run test:watch # ウォッチモード
npm run typecheck  # 型検査
```

## AIエージェントのテスト（Jest）

```bash
cd agent-server
npm test
npm run test:watch
npm run typecheck
```

## CLI のテスト（Jest、リポジトリルート）

ルートの CLI（`scripts/`）は主要プロジェクトとは別立てです。

```bash
# リポジトリルートで
npm test           # scripts/jest.config.js を使用
npm run typecheck
```

## テスト構成

| 対象 | フレームワーク | 置き場所 | テストファイル |
|---|---|---|---|
| server | Jest + Supertest | `server/__tests__/`（unit / integration / socket） | `*.test.mts` |
| agent-server | Jest | `agent-server/__tests__/`（unit / integration） | `*.test.mts` |
| client | Vitest + React Testing Library | `client/__tests__/`（111 本） | `*.test.ts(x)` |
| CLI | Jest | `scripts/__tests__/` | `*.test.ts` |

!!! note "テストはすべて `.test.mts` です"
    TypeScript 移行の途中は `.test.js` と `.test.mts` が混在していましたが、**現在 `.test.js` は 0 本**です（server / agent-server 合わせて `.test.mts` が 277 本、2026-10-15 時点）。以前このページは「混在している」と書いていましたが、古くなっていました。

    `server/jest.config.js` には移行期の設定（`.mts` を `@swc/jest`、`.js` を `babel-jest` で変換）が残っています。`jest.setup.js` などの設定ファイルは `.js` のままなので、`babel-jest` の指定は今も意味があります。

    `transform` を定義すると Jest の既定（babel-jest）が外れ、**`.js` テストの `jest.mock` 巻き上げが消える**という落とし穴があるため、`babel-jest` を明示的に復元してあります（agent-server で実際に踏んだ問題への予防）。

## テストは本番 DB につながない（[#572](https://github.com/gamasenninn/tealus/issues/572)）

server も agent-server も、テストは**テスト用の DB（`tealus_postgres_test`、5433）**に向きます。

- server は `.env.test` でテスト用 DB に向けます
- agent-server は `jest.setup.js` で**テスト用の `DATABASE_URL` を先に決めます**（dotenv は既にある変数を上書きしないので、`.env` の本番の設定に負けない）。`unit/testDbGuard.test` が、pool がテスト用 DB を指すことを見張ります

★ 以前の agent-server には同じガードが無く、2026-10-10 に作業中のテストが**本番の DB に問い合わせていました**（アプリ側は try/catch で黙るので、Postgres のログにだけ残った）。テストを書くときに DB の接続先を自分で組み立てないでください。

## CI

GitHub Actions（`.github/workflows/test.yml`）で **6 ジョブ**が走ります。すべて **Node 24** のみで、22 系以下は matrix に含まれません。

| ジョブ | 内容 |
|---|---|
| `server-test` | ライセンス確認 → 型検査 → テスト |
| `client-build` | ライセンス確認 → 型検査 → テスト → ビルド |
| `dashboard-build` | ライセンス確認 → 型検査 → ビルド |
| `agent-server-test` | ライセンス確認 → 型検査 → テスト |
| `rtc-server-typecheck` | ライセンス確認 → 型検査（mediasoup の native 依存があるためテストは走らせない） |
| `cli-test` | 型検査 → テスト（ルートの `scripts/` は他プロジェクトの依存を引かず、root の依存だけで実行） |

各ジョブで **GPL / AGPL 混入の検査**（`npm run license-fail-on-gpl`）も実行されます。
