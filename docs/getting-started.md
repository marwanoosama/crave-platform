# Getting Started

Run both services locally, then drive a full order from placement to delivery.

---

## 1. Prerequisites

| Tool | Version | Why |
| --- | --- | --- |
| Node.js | 20+ | Both services |
| PostgreSQL | 14+ | Core DB + one DB per order region |
| Redis | 6+ | Cache, agent presence, geo-search, WS pub/sub |
| RabbitMQ | 3.9+ | Core → order async events |

---

## 2. Databases

`core-service` uses a single database. `order-service` uses **one database per region**
(`eg`, `ksa` by default — see `REGIONS` in the order-service `.env`).

```sql
CREATE DATABASE quickbite_core;
CREATE DATABASE order_service_eg;
CREATE DATABASE order_service_ksa;
```

---

## 3. core-service

```bash
cd crave-core-service/backend
npm install
cp .env.example .env        # if you don't already have one — adjust DB/Redis/RabbitMQ
npm run migrate
npm run dev                 # http://localhost:3000
```

Run its outbox worker in a second terminal (this is what publishes `core.events`):

```bash
npm run worker:dev
```

Seed users, restaurants, branches, and products:

```bash
npx tsx scripts/seed.ts
```

Seeded login credentials are listed in `crave-core-service/backend/docs/TEST-USERS.md`.

---

## 4. order-service

```bash
cd crave-order-service/backend
npm install
cp .env.example .env        # adjust DB_<region>_* , Redis, RabbitMQ, CORE_SERVICE_BASE_URL
npm run migrate:all         # runs migrations on every region shard
npm run dev                 # http://localhost:4000
```

Run its workers in a second terminal — **assignment ticks and the outbox drain both
live here**, and delivery assignment will not happen without them:

```bash
npm run worker
```

> The API process serves HTTP + WebSocket only. The worker process runs the cron
> scheduler: the per-region assignment tick (every `ASSIGNMENT_TICK_SEC`, default 10s)
> and the outbound event drain.

---

## 5. Verify

```bash
curl http://localhost:3000/api/health
curl http://localhost:4000/api/health
```

The repo also ships a Bruno collection for exercising every endpoint by hand:

```
crave-order-service/backend/bruno-collection/
```

---

## 6. Drive a full order

### As a customer

1. `POST /api/auth/login` on core-service → get the `access_token` cookie.
2. `POST /api/orders` on order-service (header `X-Region: eg`, `Idempotency-Key: <uuid>`).
   - `paymentMethod: "cod"` → order lands directly in `placed`.
   - `paymentMethod: "online"` → order lands in `pending_payment` and returns a Kashier
     redirect URL.
3. For online payment, simulate the provider callback locally (the real Kashier webhook
   can't reach localhost):
   ```bash
   npx tsx play/pay-webhook.ts <orderPublicId> eg
   ```
   The order flips `pending_payment → placed`.

### As the restaurant

```bash
PATCH /api/restaurants/:restaurantId/branches/:branchId/orders/:publicId/status
{ "status": "accepted" }   # then "preparing", then "ready"
```

Each transition stamps its `<verb>_at` column, publishes a WS event to the customer and
branch rooms, and returns `409` for illegal transitions.

### As a delivery agent

1. Go online (this is what makes you a candidate):
   ```bash
   POST /api/agents/presence/online   { "lat": 30.04, "lng": 31.23 }
   ```
   Presence lives in Redis: `presence:geo:<region>` (geo set), `presence:meta:<region>:<id>`
   (TTL'd hash), `presence:busy:<region>` (set). Ping every ~30s to stay online.

2. Within ~10s the assignment worker offers you the order over WebSocket
   (`task.offered` on the `agent:<id>` room). Accept it:
   ```bash
   POST /api/agents/orders/:publicId/accept
   ```
   First acceptor wins — Redis `SETNX` plus a conditional `UPDATE` on the order row.
   The order is now `assigned` and you're marked busy.

3. Pick up and deliver:
   ```bash
   PATCH /api/agents/orders/:publicId/status   { "status": "picked" }
   PATCH /api/agents/orders/:publicId/status   { "status": "delivered" }
   ```
   `delivered` triggers the settlement transaction: commission, restaurant balance,
   and your agent earning are written atomically.

### Watching it live

Connect a Socket.IO client to `ws://localhost:4000/ws` with the access token:

```js
const socket = io("http://localhost:4000", {
    path: "/ws",
    auth: {token: accessToken},
});

socket.on("hello", (p) => console.log(p.allowedChannels));  // rooms you're allowed to join
socket.emit("subscribe", "branch:5");
socket.on("order.created", (order) => console.log(order));
```

Rooms are derived from your role: `customer:<userId>` always; `branch:<id>` and
`restaurant:<id>` for restaurant users; `agent:<userId>` for delivery agents. The server
acks each subscribe and refuses rooms you aren't entitled to.

---

## 7. Troubleshooting

| Symptom | Cause |
| --- | --- |
| Order sits in `ready`, nobody is offered it | The order-service **worker** isn't running |
| Agent is never a candidate | Presence not set — call `/agents/presence/online`, and check `PRESENCE_STALE_SEC` hasn't expired your entry |
| `409` on a status change | Illegal transition for the current status or actor — check the matrix in `docs/order-lifecycle.md` |
| `400 RegionNotResolved` | Missing `X-Region` header (or `X-Region: all` on a write) |
| `401` on internal calls | The shared `api-key` header is wrong |
| WebSocket connects but no events | You must `subscribe` to a room — `hello` only lists the permitted rooms |
