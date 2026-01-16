# Server-Sent Events Stack

実装内容:
- PostgreSQL（ローカルPostgres利用）+ Row-Level Security。
- トランザクショナルアウトボックスパターンでRedis Streamsへ書き込み。
- NestJSバックエンドがREST（`/events`）とSSE（`/sse`）を提供。
- Next.jsフロントエンドはRESTで初回50件を取得し、その後SSEで新着を購読。

## プロジェクト構成
- `backend/` – NestJSサービス + DBマイグレーション 
- `frontend/` – Next.js App Routerダッシュボード。
- `infrastructure/` – Postgres、RedisのDocker Compose。

## クイックスタート
```bash
cd infrastructure
docker compose up --build

cp backend/.env.example backend/.env

cd backend
npm install
npm run build
npm run migration:run
# seedでサンプルデータを流すようにしている。
# backend/scripts/seed.ts
npm run seed  
npm run start

cd frontend
npm install
npm run start
```

## 環境変数
ローカル起動には `backend/.env` が必要です。`backend/.env.example` をコピーして設定してください。

- `TENANT_IDS` – 受け入れるテナントIDのCSV（例: `11111111-1111-1111-1111-111111111111`）

手動検証の場合のcurl: `curl -H "x-tenant-id: <uuid>" http://localhost:3001/events?limit=50`
