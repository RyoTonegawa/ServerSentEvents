# System Design Notes

## Transactional Outbox
- **What**: `events`（本体）と `outbox`（通知待ち）を同一トランザクションで更新し、コミット後にワーカーが外部ブローカーへ publish するパターン。
- **Why**: DBへの書き込みと通知を切り離さないと「DBに書いた/書いてない」と「通知した/してない」の不整合が発生しうる。アウトボックスがあれば、データは確実に蓄積され、配信ワーカーはリトライ・再送制御に専念できる。
- **Without it**: アプリコードが `INSERT`→`publish` を順次実行するだけだと、途中障害で通知漏れや重複送信が多発し、コンポーネント間で一致を保つ仕組みを毎回実装する必要がある。

## Motivation for Decoupling Write & Notify
- **Responsibility separation**: PostgreSQL は整合性とRLSに集中し、配信レイヤー（Redis/Kafka）は低レイテンシ配信とスケールを担う。
- **プラガブルなブローカー**: ブローカーを Redis Streams から Kafka に差し替えるなど、技術選択の自由度が増す。
- **マルチシンク**: 一度アウトボックスに溜めれば、SSE、メール、Push など異なる通知チャンネルに同じイベントを配信できる。
- **課題**: ポーリングワーカーの開発/監視が必要、ブローカー運用/リトライ設計も含めた新たな複雑さが生まれる。

## Load Balancer + Pub/Sub
- **Pattern**: ロードバランサ背後に複数の SSE/WebSocket サーバーを配置し、サーバー間のイベント共有は Pub/Sub ブローカー（Redis, Kafka, etc.）経由で行う。
- **Benefit**: どのサーバーに接続しても同じイベントを受け取れる。接続数はロードバランサで水平分散でき、配信の一貫性はブローカーが保証する。
- **Relation here**: Redis Streams が Pub/Sub 的な役割を担い、SSE ブローカーはどのクライアントにも同じイベントをブロードキャストできる。WebSocket 設計で紹介されるスケールアウトパターンと同じ思考方法。

## RDB vs Message Broker Characteristics
- **RDB (PostgreSQL)**: トランザクション整合性と永続性が目的。`SELECT ... FOR UPDATE` やトランザクション維持で行ロックが発生し、ワーカーが増えるほどロック競合と IO がネックになる。
- **Message Broker (Redis Streams/Kafka)**: メモリ/ログ構造で低レイテンシ配信に特化。ブロッキング読み取りでもロック競合が少なく、複数コンシューマを前提に設計されている。
- **設計の結論**: DB でデータの正確性を担保しつつ、通知とファンアウトはメッセージブローカーへ逃がすのが合理的。RDB とブローカーは目的が異なるため、それぞれの強みを活かす。

## SSE Last-Event-ID Verification
- `EventSource` は再接続時に `Last-Event-ID` ヘッダを自動送信するため、`/sse` は `lastEventId` と `after` の両方を受け付ける。
- 動作確認: ブラウザで SSE を接続した状態でネットワークを一時的に切断→復帰し、サーバーログを確認。`requestedCursor = after || lastEventId` によって再接続後も `aggregateId` フィルタが働き、重複せずに続きから配信される。
- 手動検証: `curl -H 'Last-Event-ID: <event_id>' 'http://localhost:3001/sse?tenantId=...&after='` のようにヘッダだけ指定しても同じ結果になる（`after` を空にするとヘッダが優先される）。

## Outbox event_id vs aggregate_id
- **event_id (ULID)**: 外部公開するイベント識別子。SSE の `id:` や `Last-Event-ID`、他サービスとの交信や冪等性キーに利用する。時系列でソートできる文字列が目的。
- **aggregate_id (BIGINT)**: `events.id` (BIGSERIAL) を参照する内部 ID。REST で取得した最新 ID から SSE を再開するためのカーソルや、DB 上での順序保証に使う。
- **なぜ両方必要か**
  - `aggregate_id` のみだと、外部向けのユニーク ID (ULID) が無くなり、SSE の `Last-Event-ID` や冪等制御に不便。
  - `event_id` のみだと、DB の `events` table との結びつきが薄れ、`events.id` ベースで REST/SSE を一貫させるのが難しい（特に SQL で降順 LIMIT を取った時のカーソル計算）。
  - 役割が違うため冗長ではなく、どちらも持つことで実装がシンプルになる。
