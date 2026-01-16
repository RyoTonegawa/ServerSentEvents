# NestJS SSE バックエンド

トランザクショナルアウトボックス + Redis Streams のファンアウトを実装。

## エンドポイント
- `GET /events?limit=50` 初期取得（`x-tenant-id` が必須）。
- `POST /events` 新規イベントを挿入（body: `{ "payload": {...}, "eventType": "EventCreated" }`）。
  ```bash
  curl -X POST http://localhost:3001/events \
    -H 'x-tenant-id: 11111111-1111-1111-1111-111111111111' \
    -H 'Content-Type: application/json' \
    -d '{"payload":{"message":"hello"}}'
  ```
- `GET /sse?after=<cursor>` SSEストリーム。`Last-Event-ID` も参照します。
- Swagger UI: `http://localhost:3001/docs`

## ワーカー
`OutboxWorkerService` がテナントごとに `outbox` テーブルをポーリングし、Redis Streams に発行。SSEの購読側はストリームから読み取り、`id: <event_id>` を付けて配信する。

## コマンド
- `npm install`
- `npm run prisma:generate` (re-run when `prisma/schema.prisma` changes)
- `npm run migration:run`
- `npm run start`
- `npm test`
