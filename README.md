# MARS Cosmetics — Order & Inventory Sync Service

Design documentation for a centralized service that stops MARS Cosmetics from overselling stock across its sales channels.

> **Status:** Design only. This repository contains the problem statement and high-level design (HLD). There is no implementation code yet.

## The problem

MARS Cosmetics sells about 1,200 SKUs on its own website, Nykaa, Flipkart, and Blinkit, all fulfilled from one central warehouse. Each channel gets stock updates manually twice a day, so every channel works from stale numbers.

Example: the warehouse holds 10 units. The website takes an order for 7 and Nykaa takes an order for 5 at almost the same moment. That is 12 units promised against 10 in stock, which means overselling, cancellations, and poor marketplace ratings.

## The solution in brief

One central service that:

1. Ingests orders from all four channels (webhooks and polling) and normalizes and deduplicates them.
2. Reserves stock atomically in PostgreSQL, so two channels cannot claim the same last unit.
3. Pushes updated stock levels to each channel asynchronously, within a 60-second target under normal load.
4. Delivers confirmed orders to Microsoft Business Central (ERP) with durable, idempotent retries.

![High-level design](./images/hld.png)

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

## Known limitations

- The central database cannot stop a marketplace from accepting an order against stale stock, particularly when Flipkart polling is delayed. Timely stock pushes, conservative allocations, and reconciliation reduce the risk.
- Exactly-once ERP creation depends on Business Central supporting idempotency or reliable order lookup. This must be verified before implementation.

## Scope

**v1:** all four channel integrations, atomic reservations, cancellations and returns, async stock sync, durable ERP delivery, audit history, and a basic operations dashboard. It is sized for a three-developer team over six weeks.

**Not in v1:** multi-warehouse allocation, demand forecasting, an LLM-based investigation assistant, and a distributed event-streaming platform.

## AI disclosure

The core idea, problem analysis, and HLD are the author's own. ChatGPT was used to structure the explanation and turn the HLD into Markdown.
