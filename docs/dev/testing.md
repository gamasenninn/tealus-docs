# テスト

## サーバーテスト（Jest）

```bash
cd server

# テスト用DB起動済みであること（docker-compose up -d）
npm test           # 全テスト実行
npm run test:watch # ウォッチモード
```

テスト用DBは Docker Compose で Port 5433 に起動されます。

## クライアントテスト（Vitest）

```bash
cd client
npm test           # 全テスト実行
npm run test:watch # ウォッチモード
```

## テスト構成

- サーバー: Jest + Supertest でAPIエンドポイントのE2Eテスト
- クライアント: Vitest + React Testing Library でコンポーネントテスト
- Phase 1 リファクタリング完了時点で**130テスト**がpass確認済み
