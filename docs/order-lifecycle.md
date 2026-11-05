# Order Lifecycle

Every state an order can be in, who is allowed to move it there, and what happens as a
side effect. The authoritative implementation is
`crave-order-service/backend/src/app/order/service/order-status.service.ts` — this
document explains it.

---

## 1. States

| Status | Meaning | Who writes it |
| --- | --- | --- |
| `pending_payment` | Online order created, awaiting payment capture | system (webhook) |
| `placed` | The restaurant has the order in its queue | system (webhook / COD insert) |
| `accepted` | Restaurant accepted, cooking will start | restaurant member |
| `rejected` | Restaurant declined — terminal | restaurant member |
| `preparing` | Actively cooking | restaurant member |
| `ready` | Food ready for pickup; eligible for assignment | restaurant member |
| `assigned` | A delivery agent holds the order | assignment service / admin |
| `picked` | Agent confirmed pickup at the branch | the assigned agent |
| `delivered` | Customer received it; money settles — terminal | the assigned agent |
| `cancelled` | Cancelled before delivery | customer / restaurant / admin / system |

```
        ┌──── pending_payment ──── (online only)
        │           │
        │           │ payment captured (Kashier webhook)
        │           ▼
        └──────► placed ──► accepted ──► preparing ──► ready ──► assigned ──► picked ──► delivered
                    │            │             │             │           │            │
                    ▼            ▼             ▼             ▼           ▼            ▼
                 rejected     cancelled     cancelled    cancelled    cancelled      (terminal)
                    │
                    └──► cancelled (customer, within 60s of placed)
```

---

## 2. Transition matrix

| From \ To | accepted | rejected | preparing | ready | assigned | picked | delivered | cancelled |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `pending_payment` | | | | | | | | customer, system |
| `placed` | restaurant | restaurant | | | | | customer (≤60s), restaurant, admin |
| `accepted` | | | restaurant | | | | restaurant, admin |
| `preparing` | | | | restaurant | | | restaurant, admin |
| `ready` | | | | | system | | restaurant, admin |
| `assigned` | | | | | | agent | admin |
| `picked` | | | | | | | agent |

`assigned` and `delivered` are **not** writable through the public status endpoint —
they are written by the assignment and settlement paths.

---

## 3. Who the actors are

| Actor | Identity | Typical actions |
| --- | --- | --- |
| `customer` | JWT `role = customer`, `userId` = the order's `customer_id` | place, read, cancel (inside the window) |
| `restaurant_user` | JWT `role = restaurant_user` with `restaurantId`, `restaurantRole` (`owner` / `branch_manager` / `staff`), and `branchIds` | accept, reject, preparing, ready, cancel, read branch orders |
| `delivery_agent` | JWT `role = delivery_agent`, `userId` = the order's `delivery_agent_id` | accept/reject offers, picked, delivered |
| `system_admin` | JWT `role = system_admin` | anything, including force-assign |

RBAC permissions are owned by core-service (`orders:read`, `orders:accept`,
`orders:update`, `orders:cancel`, `payments:read`, `deliveries:assign`,
`finance:read`, `finance:payout_create`) and resolved in order-service through a cached
read-through of `core-service`.

---

## 4. Step by step

### 4.1 Placement — `POST /orders`

1. Resolve the region (header, or derived from the branch).
2. Validate the branch: active, accepting orders, restaurant `active`.
3. Validate the address belongs to the caller; snapshot its lat/lng and flat text.
4. Fetch product prices and stock in one batch; reject with `409` and the offending
   product list if anything is unavailable or under-stocked.
5. Compute money in minor units: `subtotal`, `delivery_fee`, `service_fee`, `total`.
6. Reserve stock on core-service (atomic; idempotent on the order's public id).
7. In one transaction on the region's shard: insert `orders`, insert `order_items`,
   and — for COD only — emit `order.placed` on the outbox.
8. For online orders, create the Kashier session. If that fails, void the order, release
   the stock, and surface the error.
9. Publish `order.created` to the branch room (COD only — online waits for capture).

Online orders start as `pending_payment`; COD orders start as `placed`.

### 4.2 Payment capture — `POST /payments/webhook/kashier`

1. Verify the HMAC signature; `401` if invalid.
2. De-duplicate on the provider transaction id — a replay is acknowledged and ignored.
3. On `SUCCESS`, in one transaction: mark the session `CAPTURED`, write the `charge`
   transaction, emit `payment.completed` on the outbox, and — if the order is still
   `pending_payment` — flip it to `placed` and emit `order.placed`.
4. After commit, publish `order.created` to the branch room and `order.status_changed` to
   the customer room.
5. On failure: mark the session `FAILED`, write a failed `charge` transaction, emit
   `payment.failed`. The order stays in `pending_payment` and is swept after 15 minutes.

### 4.3 Restaurant preparation

One endpoint — `PATCH /restaurants/:id/branches/:branchId/orders/:publicId/status` with
`{ "status": "..." }`. The service reads the current status, resolves the actor, and
validates the transition against the matrix. Illegal transitions return `409`.

Each accepted transition:
- stamps the matching `<verb>_at` column,
- writes the matching outbox event (`order.accepted` / `order.rejected` / `order.cancelled`)
  **in the same transaction**,
- invalidates the branch orders cache,
- publishes `order.status_changed` to the customer and branch rooms.

`rejected` and `cancelled` require a `reason` in the body.

### 4.4 Assignment

Triggered when an order reaches `ready`.

1. The **worker** (every `ASSIGNMENT_TICK_SEC`, per region) selects up to
   `ASSIGNMENT_BATCH` orders that are `ready` with no agent.
2. For each, it finds candidate agents with a Redis geo-search on `presence:geo:<region>`
   within `ASSIGNMENT_RADIUS_METERS` of the branch coordinates, keeps only agents whose
   presence metadata is still alive and who aren't already busy, and takes the closest
   `ASSIGNMENT_CANDIDATES`.
3. It broadcasts `task.offered` to each candidate's `agent:<id>` room and records the
   candidate list in Redis under `offer:order:<publicId>` (TTL
   `ASSIGNMENT_OFFER_TTL_SEC`).
4. The first agent to `POST /agents/orders/:publicId/accept` wins: a Redis `SETNX` claim
   lock arbitrates the race, and a conditional `UPDATE` on the order row guarantees the
   database agrees. The order becomes `assigned` and the agent is marked busy.
5. Losers receive `offer.cancelled`. If nobody accepts, the offer expires, the attempt
   counter increments, and the next tick re-offers — up to `ASSIGNMENT_MAX_ATTEMPTS`,
   after which an `assignment.exhausted` alert goes to the admin channel and the order
   stays `ready`.

A `POST /agents/orders/:publicId/reject` removes the agent from the candidate list.
`POST /api/admin/orders/:publicId/assign` bypasses the whole flow and force-assigns.

### 4.5 Pickup — `PATCH /agents/orders/:publicId/status { "status": "picked" }`

The service checks that the caller is the assigned agent and the order is `assigned`.
It flips the status and stamps `picked_at`, then publishes `order.status_changed` to the
customer and branch rooms. No money moves.

### 4.6 Delivery — `PATCH /agents/orders/:publicId/status { "status": "delivered" }`

One transaction settles everything:

1. Fetch the branch's commission bps (outside the transaction, from cache).
2. Compute `commission` and the agent's earning.
3. Stamp `orders.commission`.
4. For COD, write the `cod_collection` transaction as succeeded — the agent took the cash.
5. Write the `commission` transaction, and the `adjustment` transactions for the service
   fee and the delivery fee (both were paid by the customer but belong to the platform).
6. Increment the restaurant balance by `subtotal - commission`.
7. Insert the agent earning (unique per delivery → idempotent).
8. Flip the order to `delivered`, stamp `delivered_at`, emit `order.delivered` on the
   outbox.

After commit: the agent is freed from `presence:busy`, the claim lock is dropped, and
`order.status_changed` goes to the customer and branch rooms.

If anything in the transaction fails, it all rolls back — there is no partial
settlement.

---

## 5. Cancellation rules

| Who | When | Effect |
| --- | --- | --- |
| Customer | Only while `pending_payment` or `placed`, and within 60s of `placed` | Refund if captured; void the pending COD collection |
| Restaurant | Any time before `picked`, with a reason | Same |
| Admin | Any time before `picked`, with a reason | Same |
| System | `pending_payment` older than 15 minutes; assignment exhausted | Order cancelled |

Cancelling after `picked` is forbidden by the transition matrix — the food is already
with the agent.

---

## 6. WebSocket events

| Event | Room | Payload |
| --- | --- | --- |
| `order.created` | `branch:<id>` | order summary |
| `order.status_changed` | `customer:<id>`, `branch:<id>` | `{ orderId, status, ts }` |
| `task.offered` | `agent:<id>` | branch + dropoff + total + expiry |
| `task.assigned` | `agent:<id>` | the full task DTO |
| `offer.cancelled` | `agent:<id>` | `{ orderId, reason }` |
| `assignment.exhausted` | `admin:alerts` | `{ orderId, attempts }` |
