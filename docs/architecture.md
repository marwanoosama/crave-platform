# Architecture

Deep dive on how the two services fit together. Per-service design detail lives in each
repo's own `docs/` folder — this document is the map between them.

---

## 1. Bounded contexts

### core-service — "who and what exists"

| Module | Responsibility |
| --- | --- |
| `auth` | Register, login, refresh, password reset, invite acceptance. Issues the JWT both services verify. |
| `user` | User profile, current-user lookup, delivery-agent lookup (internal). |
| `restaurant` | Restaurant CRUD + lifecycle status (`active` / `suspended`). |
| `branch` | Branch CRUD, geo/nearby search, branch status, accept-orders flag, delivery fee, commission bps. |
| `product` | Categories and products, availability, price, stock; atomic reserve/release stock. |
| `rbac` | Roles, permissions, restaurant members and their branch access. |
| `customer-address` | Saved delivery addresses for customers. |

It is **read-mostly** — high read traffic, low write traffic, and the data is
*reference* data that changes slowly.

### order-service — "what happened and what it cost"

| Module | Responsibility |
| --- | --- |
| `order` | Order placement, the status machine, customer/restaurant reads, core-data cache. |
| `payment` | Kashier sessions, webhooks, transactions (charge / cod_collection / commission / adjustment). |
| `agent` | Agent presence, task list, earnings, and the delivery settlement transaction. |
| `assignment` | The auto-assignment worker and the admin force-assign override. |
| `finance` | Restaurant balances and payouts. |

It is **write-heavy and money-critical**. Every state change is a transaction, and every
money movement is a ledger row.

---

## 2. Data model

### core-service

```
users ─┬─ customer_addresses
       ├─< restaurants ──< restaurant_branches ──< products
       │                          └──< branch_products (stock)
       └─< restaurant_members >── roles ──< permissions
```

### order-service (per region shard)

```
orders ─┬─< order_items
        ├─< transactions          (charge | cod_collection | commission | adjustment)
        ├─< payment_sessions
        └─< agent_earnings

restaurant_balances          (one row per restaurant + currency)
payment_webhook_events       (webhook de-duplication ledger)
events_outbox                (transactional outbox)
idempotency_keys             (durable idempotency for critical writes)
```

Cross-service references (`users.id`, `restaurants.id`, `products.id`) are **logical
only** — there is no database-level foreign key across services. The order-service
resolves them through the `core-client` HTTP wrapper with a read-through cache.

---

## 3. Sharding

`order-service` shards Postgres by **country** (`region` in code and columns, so the
router stays generic if sub-sharding is ever needed). `core-service` is not sharded.

| Candidate | Locality | Hot-spot risk | Cross-shard reads |
| --- | --- | --- | --- |
| `customer_id` | poor | low | restaurant's order list → fan-out |
| `restaurant_id` | poor | very high (big chains) | customer history → fan-out |
| `order_id` | none | low | every read is a fan-out |
| **country** | **high** | medium (meal-hour peaks) | rare — admin global views only |

A customer, the restaurant they order from, and the courier who delivers all live in the
same country, so `country` gives real locality for all three actors.

**Region resolution:** the `X-Region` header. `X-Region: all` is allowed for admin
fan-out reads; write endpoints require a concrete region. `POST /orders` derives the
region from the chosen branch's cached metadata when the header is absent.

**Mitigations for meal-hour peaks:** Redis caching of the per-branch "pending orders"
list, horizontal scaling of the stateless API tier per region, and (deferred) read
replicas.

---

## 4. Communication

### Synchronous: order-service → core-service

Over HTTP with a shared `api-key` header (`requireInternalApiKey`). Every call goes
through the `core-client` wrapper, which adds correlation ids, retries, and a
read-through Redis cache.

| Call | Cache key | TTL |
| --- | --- | --- |
| `GET /internal/branches/:id` | `core:branch:<id>` | 60s |
| `GET /internal/branches/:id/products` | `core:product:price:<branchId>:<productId>` | 30s |
| `GET /internal/restaurants/:id` | `core:restaurant:<id>` | 5m |
| `GET /internal/rbac/permissions` | `core:rbac:perms:<role>` | 5m |

Cache entries are invalidated by the inbound RabbitMQ events below — not by TTL alone.

### Asynchronous: core-service → order-service

RabbitMQ topic exchange `core.events`, durable queue `order-service.core-events`, with a
dead-letter queue for poison messages.

| Event | Effect in order-service |
| --- | --- |
| `product.stock.changed` | Invalidate `core:product:*` cache entries |
| `product.price.changed` | Invalidate `core:product:*` cache entries |
| `branch.updated` | Invalidate `core:branch:*` |
| `branch.deactivated` | Invalidate `core:branch:*` + start rejecting new orders for that branch |
| `restaurant.suspended` | Invalidate `core:restaurant:*` + flag pending orders for review |
| `rbac.permissions_changed` | Invalidate `core:rbac:perms:*` |

Delivery is **at-least-once** with manual ack after the handler commits; duplicates are
de-duplicated via a Redis `SETNX` on the event id. Every handler is an idempotent cache
invalidation, so expiring the dedupe key is safe.

### Outbound: order-service → consumers

Both services use the same **transactional outbox** pattern: the domain change and its
outbox row are written in the same database transaction, and a separate worker drains
the table with `FOR UPDATE SKIP LOCKED` and publisher confirms.

`order-service` emits: `order.placed`, `order.accepted`, `order.rejected`,
`order.delivered`, `order.cancelled`, `payment.completed`, `payment.failed`.

### Realtime: order-service → clients

Socket.IO on `/ws`, with the Redis adapter so any API instance can fan out to a socket
connected to any other instance.

| Room | Who joins | Receives |
| --- | --- | --- |
| `customer:<userId>` | the customer | `order.status_changed`, `order.created` |
| `branch:<branchId>` | restaurant staff | `order.created`, `order.status_changed` |
| `restaurant:<restaurantId>` | restaurant staff | order events |
| `agent:<userId>` | the delivery agent | `task.offered`, `task.assigned`, `offer.cancelled` |

The server derives the permitted rooms from the JWT at handshake time and rejects
subscriptions to anything else.

---

## 5. Money model

Everything is an `INT` of **minor units** (piasters/cents) with a `currency` column.
No decimals, no floats — integer math is exact and aggregations stay fast.

```
subtotal + delivery_fee + service_fee = total
commission = floor(subtotal × commissionBps / 10000)
agent_earning = floor(delivery_fee × agentEarningShareBps / 10000)
restaurant_balance += subtotal - commission
```

Money movement is modelled as rows in `transactions`, typed by `transaction_type`:

| Type | Direction | When |
| --- | --- | --- |
| `charge` | customer → restaurant owner | online payment captured, or COD collected on delivery |
| `cod_collection` | customer → restaurant owner | cash handed to the agent |
| `commission` | restaurant owner → platform | settlement |
| `adjustment` | restaurant owner → platform | service fee and delivery fee claw-back |

**Settlement is one transaction.** On `delivered`: commission is stamped, the COD/charge
transaction is written, adjustments are booked, the restaurant balance is incremented, the
agent earning is inserted, and the order flips to `delivered` — all together, or not at
all. There is no partial settlement state.

---

## 6. Reliability primitives

| Concern | Mechanism |
| --- | --- |
| Duplicate order creation | `Idempotency-Key` header, Redis + durable `idempotency_keys` table, 24h TTL, body-hashed (same key + different body → `409`) |
| Duplicate webhook delivery | De-duplicated on the provider transaction id at the SQL boundary |
| Concurrent assignment | Redis `SETNX` claim lock + conditional `UPDATE orders … WHERE status='ready' AND delivery_agent_id IS NULL` |
| Lost events | Transactional outbox + `FOR UPDATE SKIP LOCKED` drain + publisher confirms |
| Stock oversell | Atomic `SELECT … FOR UPDATE` decrement in core-service, `409` on underflow |
| Event published for a rolled-back state | Outbox row is written inside the same transaction as the state change |

---

## 7. Performance rules both services follow

- No N+1 queries — repositories batch with `whereIn` or join.
- Every query is backed by an index; migrations carry a comment naming the query it supports.
- No `SELECT *` — explicit column lists.
- Cursor pagination, never offset.
- Read-heavy endpoints are cached with explicit invalidation on write, not TTL alone.
- Hot writes (order insert, payment status update) target < 200ms p95.
