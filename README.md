# Crave — Food Delivery Platform

Two production-shaped microservices that together implement a food-delivery platform:
customers order from restaurants, pay online or cash-on-delivery, and delivery agents
pick up and deliver the food — with live updates pushed to every client app.

This repo is the **system overview**. The code lives in two sibling repositories:

| Service | Repo | Role |
| --- | --- | --- |
| `core-service` | [marwanoosama/crave-core-service](https://github.com/marwanoosama/crave-core-service) | Identity, restaurants, branches, catalog, RBAC — the **read-mostly** side. |
| `order-service` | [marwanoosama/crave-order-service](https://github.com/marwanoosama/crave-order-service) | Orders, payments, deliveries, agent assignment, restaurant finance, WebSocket — the **write/money** side. |

---

## Architecture

```
        [Customer App]   [Restaurant Dashboard]   [Delivery App]   [Admin Dashboard]
               \                |                       |                  |
                \               v                       v                 /
                 +---------->  API gateway  ----------------------------+
                                     |
                 +-------------------+-------------------+
                 |                                       |
                 v                                       v
        +------------------+                    +------------------+
        |   core-service   |  sync HTTP (api-key) |   order-service  | <---- WebSocket
        |  :3000           |<------------------->|  :4000           |       to all 4 apps
        |                  |  async via RabbitMQ |                  |
        | users            |  (cache invalid.)  | orders            |
        | auth / JWT       |                    | payments (Kashier)|
        | restaurants      |                    | deliveries        |
        | branches         |                    | agent assignment  |
        | products         |                    | finance           |
        | RBAC             |                    | websocket hub     |
        +------------------+                    +------------------+
                 |                                       |         |
                 v                                       v         v
            Postgres                              Postgres     Redis
        (quickbite_core)                    (sharded per country)  (cache +
                                                   eg / ksa       presence +
                                                                    geo + pubsub)
                                                                     |
                                                                     v
                                                        Kashier (payment provider)
                                                      sync init / async webhook
```

### Service boundary

`core-service` owns **who and what exists**; `order-service` owns **what happened and
what it cost**. The split is deliberate:

- **Different write profile.** Menu browsing is read-heavy; checkout is high-write and
  money-critical. They scale and fail independently.
- **Isolation.** Money and order data live in their own clusters — a noisy catalog query
  cannot starve checkout.
- **Sharding fits orders, not the catalog.** A customer, a restaurant, and a courier are
  all in the same country, so `country` is the natural shard key for orders. The global
  product catalog does not shard that way.
- **Failure domain.** A `core-service` incident degrades order placement gracefully
  (cached branch/product metadata) instead of taking it down.

---

## The order lifecycle (end to end)

```
pending_payment ──► placed ──► accepted ──► preparing ──► ready
   (online only)        │                                          │
                        │                                          ▼
                        └──► rejected / cancelled              assigned
                                                                   │
                                                                   ▼
                                                               picked
                                                                   │
                                                                   ▼
                                                              delivered  (settlement)
```

1. **Place** — `POST /orders` validates the branch, address, and product stock against
   `core-service`, computes money in minor units, reserves stock atomically, and inserts
   the order in a single transaction.
2. **Pay** — online orders create a Kashier session and wait in `pending_payment`. The
   signed webhook flips the order to `placed`. COD orders start at `placed`.
3. **Prepare** — restaurant staff move the order through `accepted → preparing → ready`
   on one `PATCH /orders/{id}/status` endpoint guarded by a transition matrix.
4. **Assign** — a background worker polls ready orders, finds online agents nearby via
   Redis geo-search, and broadcasts a `task.offered` over WebSocket. The first agent to
   accept wins the order (Redis `SETNX` + a conditional SQL `UPDATE`).
5. **Deliver** — the agent marks `picked`, then `delivered`. Delivery triggers the money
   settlement transaction: commission, restaurant balance, and agent earnings, all in one
   atomic database transaction.

Live status changes fan out over Socket.IO to `customer:<id>`, `branch:<id>`, and
`agent:<id>` rooms.

---

## Cross-service communication

| Direction | Mechanism | Used for |
| --- | --- | --- |
| order → core | **sync HTTP** + shared `api-key` | branch/product/address lookups, RBAC permission resolution, stock reserve/release |
| core → order | **async RabbitMQ** (`core.events` topic exchange) | `product.stock.changed`, `product.price.changed`, `branch.updated`, `branch.deactivated`, `restaurant.suspended`, `rbac.permissions_changed` → cache invalidation in order-service |
| order → clients | **WebSocket** (Socket.IO + Redis adapter) | order status, task offers, live agent positions |
| order → Kashier | sync session create + **async webhook** | online payments and refunds |
| order → analytics | **transactional outbox** → RabbitMQ | `order.placed`, `order.accepted`, `order.rejected`, `order.delivered`, `order.cancelled`, `payment.completed`, `payment.failed` |

Both services write domain changes and their outbox row **in the same database
transaction**, so a published event can never describe a state that was rolled back.

---

## Key technical decisions

- **Region sharding.** `order-service` shards Postgres by country (`eg`, `ksa`). The
  shard router resolves the region from the `X-Region` header; `POST /orders` derives it
  from the branch when absent. Cross-shard reads are admin-only and capped.
- **Money as integers.** Every amount is stored as `INT` minor units (piasters/cents)
  with a `currency` column — never `DECIMAL`, never floats.
- **Transactional outbox.** Events are written inside the domain transaction and drained
  by a worker with `FOR UPDATE SKIP LOCKED` + publisher confirms, so N workers can run
  concurrently without duplicate publishes.
- **Idempotency.** Order creation, payment init, and assignment endpoints require an
  `Idempotency-Key`. Same key + same body replays the original response; same key +
  different body returns `409`.
- **Idempotent webhooks.** Kashier webhooks are de-duplicated on the provider transaction
  id at the SQL boundary, so retries are safe.
- **Conditional claims.** Assignment is won with `UPDATE … WHERE status='ready' AND
  delivery_agent_id IS NULL` — the database, not application logic, arbitrates the race.
- **Redis geo-assignment.** Agent presence lives entirely in Redis (geo set + TTL'd
  metadata + busy set); the hot assignment path never touches Postgres.
- **Response DTOs.** Every HTTP response is shaped by a dedicated DTO class, so column
  changes never leak into the wire format.

---

## Repository layout

```
crave-platform/          ← you are here — overview only
crave-core-service/      ← backend/src/app/{auth,user,restaurant,branch,product,rbac,customer-address}
crave-order-service/     ← backend/src/app/{order,payment,agent,assignment,finance}
```

Both services are Node.js + TypeScript (strict, decorators on), Express 5, `knex` over
`pg`, `tsyringe` for DI, `ioredis` for cache/presence/pubsub, `amqplib` for RabbitMQ,
Socket.IO for realtime, and Zod-validated env config.

---

## Documentation

| Doc | Contents |
| --- | --- |
| [`docs/getting-started.md`](docs/getting-started.md) | Run both services locally, migrate, seed, and drive a full order through to delivery. |
| [`docs/architecture.md`](docs/architecture.md) | Service boundaries, data model, sharding, event topology, money model. |
| [`docs/order-lifecycle.md`](docs/order-lifecycle.md) | Every state transition, who is allowed to make it, and the side effects. |

Deep-dive docs live in each service repo:

- `crave-order-service/backend/docs/` — system design, database design, API contracts,
  folder structure, PRD, and a per-module `business-logic/` folder.
- `crave-core-service/backend/docs/TEST-USERS.md` — seeded login credentials.

---

## API surface at a glance

**core-service** (`:3000`)

```
POST   /api/auth/{register,login,refresh,forget-password,reset-password,accept-invite}
GET    /api/user/me                       PATCH /api/user/me
GET    /api/restaurants                   POST /api/restaurants
GET    /api/restaurants/:id               PATCH /api/restaurants/:id  PATCH /api/restaurants/:id/status
GET    /api/restaurants/:id/branches      POST /api/restaurants/:id/branches
GET    /api/branches/nearby               PATCH /api/branches/:id     PATCH /api/branches/:id/status
GET    /api/restaurants/:id/products      POST /api/restaurants/:id/products
GET    /api/branches/:id/products         GET /api/products/:id      PATCH /api/products/:id
GET    /api/customer/addresses            POST/PATCH/DELETE /api/customer/addresses/:id
GET    /api/roles/:role/permissions
GET/POST/PATCH/DELETE /api/restaurants/:id/members[/:memberId][/branches]
# internal (shared api-key): /api/internal/branches, /api/internal/branches/:id,
#   /api/internal/branches/:id/products, /api/internal/branches/:id/{reserve,release}-stock,
#   /api/internal/customer/addresses/:id, /api/internal/rbac/permissions, /api/internal/agents/:id
```

**order-service** (`:4000`)

```
POST   /api/orders                        GET /api/orders/:publicId
GET    /api/customer/orders               PATCH /api/customer/orders/:publicId/status
GET    /api/restaurants/:id/branches/:branchId/orders
PATCH  /api/restaurants/:id/branches/:branchId/orders/:publicId/status
PATCH  /api/admin/orders/:publicId/status
POST   /api/payments/webhook/kashier      GET /api/restaurants/:id/payments/:paymentId
POST   /api/agents/presence/{online,ping,offline}
POST   /api/agents/orders/:publicId/{accept,reject}
PATCH  /api/agents/orders/:publicId/status        GET /api/agents/{tasks,earnings}
POST   /api/admin/orders/:publicId/assign          (force-assign an agent)
GET    /api/restaurants/:id/{balance,payouts}      POST /api/admin/restaurants/:id/payouts
WS     /ws  → rooms: customer:<id>, branch:<id>, agent:<id>
```

---

## Notes

- **Region header.** `order-service` requires `X-Region` on most endpoints
  (`X-Region: all` is allowed for admin fan-out reads; writes need a concrete region).
- **Auth.** Both services verify the same JWT shape (`userId`, `role`, `restaurantId`,
  `restaurantRole`, `branchIds`) — access token in the `access_token` cookie or a Bearer
  header; Socket.IO also accepts `auth.token`.
- **Online payments are region-gated.** `ONLINE_PAYMENT_REGIONS=eg` — other regions fall
  back to COD.
