# MARS Cosmetics — Order & Inventory Sync

## Recommendation

Use PostgreSQL as the single authority for sellable stock, a durable inbox/outbox, and independent workers for reservations, channel updates and ERP delivery. Short per-SKU transactions prevent **internal double-reservation**; queues isolate channel and ERP outages.

**Limit:** four independent marketplaces cannot be atomically locked by this service. Allocations, buffers, fast updates and reconciliation reduce overselling, but the ≤0.5% target is not a strict guarantee unless every channel enforces its published quantity and ERP writes are idempotent.

**Diagram:** [PNG](architecture.png) · [editable Mermaid source](architecture.mmd)

## Architecture and data

NestJS webhook endpoints accept website, Nykaa and Blinkit events; a Flipkart poller runs every two minutes with a durable cursor. The API validates an event, writes `inbox_events` and an outbox row in one PostgreSQL transaction, then acknowledges it. An outbox relay publishes to separate SQS order, stock and ERP queues. This avoids a database/message dual-write; consumers still handle duplicate delivery. [AWS transactional outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) · [SQS delivery](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html)

Run Node/NestJS services on ECS, RDS PostgreSQL, SQS + DLQs, and a small Redis deployment. ECS workers autoscale from queue backlog/age with hard spend caps. Redis coordinates only per-channel rate limits—not stock truth—using an atomic token bucket. [ECS queue scaling](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-autoscaling-queue.html) · [Redis rate limiter](https://redis.io/docs/latest/develop/use-cases/rate-limiter/nodejs/)

| Table | Important keys / constraints |
|---|---|
| `inventory` | `sku` PK; `on_hand`, `reserved`, `version`; check `0 ≤ reserved ≤ on_hand`; available = `on_hand - reserved`. |
| `orders`, `order_lines` | Unique `(channel, external_order_id)`; lines keyed by `(order_id, sku)`; retain normalized status and source revision. |
| `inbox_events` | Unique `(channel, source_event_id)`; source version, payload hash, receipt time and processing state. |
| `inventory_ledger` | Append-only SKU, delta, reason, order/event, time and actor; unique event/reason prevents double release. |
| `channel_allocations` | `(sku, channel)`; allocated/desired quantity, applied version and sync status; total allocation ≤ sellable stock. |
| `outbox`, `erp_deliveries`, `sync_attempts` | Durable payload/status; unique ERP logical key; attempts, retry time, error and external document ID. |

For each reservable order, one short transaction deduplicates it, locks SKU rows in sorted order, checks availability, increments `reserved`, reduces that channel’s allocation, writes ledger rows and enqueues effects. PostgreSQL `FOR UPDATE` serializes writers on the same row; consistent lock order reduces deadlocks. [PostgreSQL locking](https://www.postgresql.org/docs/current/explicit-locking.html) Database unique constraints—not an application check-then-insert—arbitrate duplicates. [PostgreSQL constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)

**Last-unit race:** with one unit left and simultaneous Nykaa/website sales, the first transaction locks the SKU, reserves the unit and commits. The second waits, sees zero available, and becomes `allocation_failed`; never let stock go negative. Escalate/cancel with an audit entry. Allocations whose sum is at most sellable stock limit exposure beforehand. If a marketplace accepts beyond its last published quantity or during an in-flight reduction, flag the mismatch and push zero; the service cannot reverse that external sale.

## Events, stock sync and failure handling

Deduplicate by `(channel, external_order_id)` plus stable event ID/source revision. Apply only newer revisions; reconcile current channel status when revisions are absent, and retain cancel-before-create tombstones. Without revisions or status lookup, strict out-of-order correctness is an assumption. Flipkart polling uses an overlapping cursor window and the same inbox.

Publish only the newest **absolute** quantity per `(SKU, channel)`; skip stale queued versions. Redis token buckets enforce Nykaa ≤5 one-SKU calls/sec and Flipkart ≤10 calls/sec in batches of up to 50; back off on 429. Blinkit’s daily CSV is the snapshot path and REST is for urgent changes. On timeout, retry an absolute set only if idempotent; otherwise read back/reconcile. Website updates use its owned API/DB adapter.

Target p95 propagation under 60 seconds at normal load, prioritizing reductions/zero. Nykaa can handle 300 one-SKU calls/minute; more than 300 distinct changed SKUs/minute cannot fit the 60-second window. The order peak alone does not state distinct SKUs, so load-test and alert on backlog rather than promise the impossible.

ERP downtime does not block intake or stock pushes. Keep confirmed orders queued; pause retries during Sunday maintenance, then retry with exponential backoff/jitter. Move poison messages to a DLQ after bounded attempts; show errors/history in the ops retry screen and audit replays. [SQS DLQs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)

For effectively-once ERP effects, Business Central must atomically store a stable key such as `channel:external_order_id` with the order. Standard `salesOrder` exposes `externalDocumentNumber`, but docs do not promise idempotent POST; a custom API/table can provide the needed extension point. [Sales order API](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/api-reference/v2.0/resources/dynamics_salesorder) · [Custom APIs](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-develop-custom-api) · [Idempotent consumers](https://learn.microsoft.com/en-us/azure/architecture/patterns/idempotent-consumer). Without a BC-side unique key, reconcile an ambiguous timeout before retrying; hard exactly-once effects are not provable.

## Operations, scope and assumptions

Day-one alerts: event-to-reservation lag/duplicate count; stock-push p95 age, 429/timeouts and backlog; ERP oldest-message age/failures/DLQ; allocation failures and stock-cancellation rate vs 0.5%; DB lock waits/deadlocks/connections. Ops filters failed syncs by channel/order/SKU, sees last error/next retry and audits each replay.

V1: one warehouse, four channels, order/reservation/cancel/restock events, BC delivery, audit/retry UI and stock reconciliation. Defer multi-warehouse, predictive allocation, dynamic pricing, LLM assistant and full returns-quality workflow.

Assumptions: (1) warehouse/ERP feed or controlled import seeds `on_hand` and supplies receipts, adjustments and picks; (2) channels provide stable order IDs plus revisions/event IDs or status lookup, and enforce absolute listing quantities; (3) `ORDER_CREATED` reserves immediately, only confirmed/paid orders go to ERP, and returns restock after warehouse confirmation; (4) BC can enforce a unique integration key through an extension/custom API; (5) AWS Mumbai is used, and Multi-AZ RDS, Redis, ECS, NAT and log costs are validated against ₹50,000/month before provisioning.

## Trade-offs

| Choice | Rejected | Reason |
|---|---|---|
| PostgreSQL row locks + ledger | Redis locks / optimistic-only stock | Durable arbiter and audit; hot SKUs serialize, but transactions stay short. |
| Outbox + SQS standard | Synchronous fan-out / Kafka or FIFO everywhere | Low-ops fit and outage isolation; inbox keys/revisions handle duplicates and reordering. |
| Per-channel allocations + buffer | Publish full global quantity everywhere | Reduces stale-snapshot risk, but may strand stock and needs tuning. |
| BC unique key/custom API | Blind retries to standard POST | Makes ambiguous retries safe; requires a small ERP extension. |

**AI-use note:** Cue assistant and official vendor documentation were used for research and drafting; no code or real channel/ERP APIs were used.
