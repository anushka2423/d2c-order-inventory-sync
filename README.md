# D2C — Order & Inventory Sync Service

Design documentation for a centralized service that stops D2C from overselling stock across its sales channels.

> **Status:** Design only. This repository contains the problem statement and high-level design (HLD). There is no implementation code yet.

## The problem

D2C sells about 1,200 SKUs on its own website, Nykaa, Flipkart, and Blinkit, all fulfilled from one central warehouse. Each channel gets stock updates manually twice a day, so every channel works from stale numbers.

Example: the warehouse holds 10 units. The website takes an order for 7 and Nykaa takes an order for 5 at almost the same moment. That is 12 units promised against 10 in stock, which means overselling, cancellations, and poor marketplace ratings.

## The solution in brief

One central service that:

1. Ingests orders from all four channels (webhooks and polling) and normalizes and deduplicates them.
2. Reserves stock atomically in PostgreSQL, so two channels cannot claim the same last unit.
3. Pushes updated stock levels to each channel asynchronously, within a 60-second target under normal load.
4. Delivers confirmed orders to Microsoft Business Central (ERP) with durable, idempotent retries.

![High-level design](./images/hld.png)

## Setup

There is nothing to build or run yet. The repo is documentation only, so setup means getting the docs open.

1. Clone the repository and enter it:
   ```bash
   git clone <repo-url>
   cd d2c-order-inventory-sync
   ```
2. Read [design.md](./design.md) in any Markdown viewer (GitHub, VS Code preview, etc.). It references the images in [images/](./images), so keep the folder structure intact.
3. Start with the [approach](#approach) below for the short version, then follow the section pointers under [Where to look in design.md](#where-to-look-in-designmd).

### Planned local environment (for implementation)

When implementation begins, the design implies these local dependencies. Exact versions, language, and framework are not decided yet.

| Dependency | Why |
|---|---|
| PostgreSQL | Source of truth for orders, inventory, and the job outbox |
| Redis | Caching and coordination only, never the authoritative stock store |
| Sandbox or mock endpoints for Nykaa, Flipkart, Blinkit, and Business Central | Develop against channel rate limits and failure modes without touching production |

Suggested bootstrap order follows [Implementation Priorities](./design.md#12-implementation-priorities): schema and migrations first, then ingestion and atomic reservation, then workers.

## Approach

The design starts from one observation: overselling happens because several systems each hold their own copy of "available stock" and decide independently. The fix is to make exactly one place decide, and make every other system a follower.

### 1. One authority for stock

PostgreSQL holds the only authoritative inventory record per SKU, with `stock` and `reserved` quantities. Channels never decide whether stock exists; they only send orders and receive stock levels.

### 2. Reserve atomically, not read-then-write

A read-then-update lets two orders both see "1 unit left". Instead, the reservation is a single conditional update inside a transaction, along the lines of:

```sql
UPDATE inventory
SET reserved = reserved + :qty
WHERE sku = :sku AND stock - reserved >= :qty;
-- 0 rows updated => insufficient stock, reject the order
```

Multi-item orders reserve inside one transaction and roll back entirely if any line fails. The same transaction writes an `inventory_movements` audit row and an `outbox_jobs` row, so the stock change and the work it triggers are committed together or not at all.

### 3. Make ingestion idempotent

Webhooks repeat, arrive out of order, and polling overlaps. Every event is recorded in `inbound_events` and keyed on `(channel, external_order_id)`, so a replay cannot create a second order or reserve stock twice. Flipkart polling uses checkpoints; Blinkit adds reconciliation where an API or report allows it.

### 4. Do slow work asynchronously

Calls to channels and the ERP happen in background workers reading the `outbox_jobs` table, never inside the order request. A slow or down integration cannot block orders from other channels. Stock updates are coalesced so workers send the latest desired quantity rather than every intermediate one, and each channel is driven within its own limits (Nykaa one SKU per request at 5 req/s, Flipkart batches of 50, Blinkit CSV plus REST for urgent changes).

### 5. Assume failure and reconcile

- Retries are bounded, with exponential backoff and jitter, and honor `Retry-After`.
- After a timeout the system checks what actually happened before retrying, since the provider may have processed the request.
- Jobs that keep failing are flagged for manual review and can be retried without creating a new order or reservation.
- ERP delivery is one record per order with a stable reference, so retries do not duplicate orders.

### 6. Stay small and observable

A single PostgreSQL database and a handful of workers on a small AWS footprint, instead of an event-streaming platform, fit a three-developer, six-week v1. Metrics cover ingestion lag, stock sync latency, reservation failures, ERP backlog, and integration errors, with an operations dashboard for failed syncs and inventory history.

For the reasoning behind each choice and the alternatives rejected, see [Trade-offs](./design.md#11-trade-offs).

## Key design decisions

| Decision | Approach |
|---|---|
| Inventory source of truth | PostgreSQL transactions with conditional updates, not Redis |
| Processing model | Durable jobs (outbox pattern) handled by background workers, not synchronous channel calls |
| Reliability | Deduplicated inbound events, bounded retries with exponential backoff and jitter, reconciliation after timeouts |
| Deployment | Small AWS footprint with managed PostgreSQL, not a microservices platform |

## Repository contents

| Path | Description |
|---|---|
| [design.md](./design.md) | Full design: HLD, data model, ingestion, consistency, stock sync, failure handling, observability, scope, assumptions, trade-offs |
| [images/problem_statement.png](./images/problem_statement.png) | Sketch of the overselling problem |
| [images/hld.png](./images/hld.png) | High-level architecture diagram |

## Where to look in design.md

- **Preventing overselling:** section 5, Inventory Consistency
- **Per-channel behavior:** section 4, Order Ingestion, and section 6, Stock Synchronization
- **What happens when things break:** section 7, Failure Handling
- **v1 scope and build order:** sections 9 and 12


## AI disclosure

The core idea, problem analysis, and HLD are the author's own. ChatGPT was used to structure the explanation and turn the HLD into Markdown.
