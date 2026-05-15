# MFQ Test Plan – E-commerce System

## M – Models

### M1. End-to-End Business Flow Model
Browse → Add to Cart → Place Order → Reserve Inventory → Payment → Deduct Inventory → Shipment → Sign → (Return → Restore Inventory)

### M2. Order State Machine

| State     | Allowed Events        | Next State   | Conditions / Actions                          |
|-----------|-----------------------|--------------|-----------------------------------------------|
| Pending   | Pay                   | Paid         | Payment success, deduct inventory async       |
| Pending   | Cancel                | Cancelled    | Release reserved inventory                    |
| Paid      | Ship                  | Shipped      | WMS notified, inventory sold                  |
| Paid      | Cancel                | Cancelled    | Refund + restore inventory                    |
| Shipped   | Sign                  | Completed    | Customer signed                               |
| Shipped   | Return                | Returned     | After inspection, restore inventory           |
| Completed | Return(within window) | Returned     | Restore inventory                             |

### M3. Inventory State Model
- **Available** → `reserve()` → **Reserved** (order created)  
- **Reserved** → `deduct()` → **Sold** (payment & shipment)  
- **Reserved** → `release()` → **Available** (cancel / timeout)  
- **Sold** → `restore()` → **Available** (return)  
- All transitions are **idempotent** and **logged**.

### M4. External Interaction Model
- Storefront ↔ WMS / SC2P
- Sync: Order, Shipment, Return, Inventory
- Mode: Async Message + Sync Callback + Idempotent
- **Sync types**:  
  - Reserve: synchronous (strong consistency)  
  - Deduct: asynchronous (message queue)  
  - WMS stock sync: batch every 15 minutes

### M5. Consistency Boundaries

| Operation                     | Consistency Type       | Reconciliation Point                     |
|-------------------------------|------------------------|------------------------------------------|
| Reserve on order creation     | Strong (DB transaction)| N/A                                      |
| Deduct after payment          | Eventual (MQ)          | Hourly diff check (Inventory vs WMS)    |
| Cancel restore                | Strong + Compensation  | Daily reserved vs sold check             |
| Return restore                | Eventual (WMS callback) | Daily inventory reconciliation          |

---

## F – Functional

### F1. Core Happy-Path Coverage (P0)

| ID    | Scenario                             | Expected Result                                    |
|-------|--------------------------------------|----------------------------------------------------|
| FP-01 | Place order with available stock     | Order created, inventory reserved, status Pending  |
| FP-02 | User pays successfully               | Status = Paid, inventory deducted, WMS notified    |
| FP-03 | Admin triggers shipment              | Status = Shipped, tracking number generated        |
| FP-04 | Customer signs for delivery          | Status = Completed                                 |
| FP-05 | Customer returns within policy       | Status = Returned, inventory restored              |

### F2. Inventory Sync Scenarios (Mandatory)

| ID    | Scenario                                      | Priority | Expected Result                                |
|-------|-----------------------------------------------|----------|------------------------------------------------|
| IS-01 | Reserve success – stock available             | P0       | Available -1, Reserved +1                      |
| IS-02 | Reserve failure – insufficient stock          | P0       | Order rejected, no reservation                 |
| IS-03 | Deduct success after payment                  | P0       | Reserved -1, Sold +1, WMS async call sent      |
| IS-04 | Deduct failure (WMS down)                     | P1       | Order stays Paid, retry+backoff, alert         |
| IS-05 | Cancel order – restore inventory              | P0       | Reserved -1, Available +1                      |
| IS-06 | Idempotency – duplicate payment callback      | P1       | Deduct executed once, second ignored           |
| IS-07 | Out of order message: Deduct before Reserve   | P1       | Message rejected or queued, no negative stock  |
| IS-08 | Restore after return                          | P1       | Sold -1, Available +1, sync to WMS             |

### F3. Boundary Scenarios

| ID    | Scenario                                      | Priority | Expected Result                                |
|-------|-----------------------------------------------|----------|------------------------------------------------|
| BS-01 | Zero stock order attempt                      | P1       | Order fails with "Out of stock"                |
| BS-02 | Flash sale (1000 req for 10 items)            | P0       | Only 10 succeed, rest "Sold out"               |
| BS-03 | Partial shipment (2 SKUs, 1 unavailable)      | P2       | Ship available part, backorder rest            |
| BS-04 | Split shipment from multiple warehouses       | P2       | Multiple shipment records, inventory per WH    |
| BS-05 | Return with damaged item                      | P2       | Stock restored only for sellable items         |

### F4. Critical API Validations

| API                    | Field Validation                      | Error Codes                     | Callback / Writeback                       |
|------------------------|---------------------------------------|---------------------------------|--------------------------------------------|
| Create Order           | productId, quantity, userId           | 400 (invalid), 409 (conflict)   | Returns orderId                            |
| Payment Callback       | orderId, amount, transactionId        | 200, 400, 409 (duplicate)       | Updates order status, deduct inventory     |
| Cancel Order           | orderId, reason                       | 200, 404, 409 (wrong state)     | Restores inventory                         |
| WMS Shipment Callback  | orderId, trackingNumber, skus         | 200, 400                        | Updates order status to Shipped            |

---

## Q – Quality

### Q1. Performance

| Metric               | Target                          | Conditions                     |
|----------------------|---------------------------------|--------------------------------|
| Concurrent users     | 10,000                          | Flash sale scenario            |
| Order create API     | P99 < 200ms                     | 5000 rpm                       |
| Payment callback     | P99 < 100ms                     | 3000 rpm                       |
| Error rate           | < 0.1%                          | Under target load              |
| Throughput           | > 500 orders/sec                | With reservation + async deduct|

### Q2. Reliability

- **Retry**: Deduct failure → 3 retries (1s, 2s, 4s) → dead letter queue (DLQ).  
- **Fallback**: WMS unavailable → local queue + retry, user sees "Order confirmed, shipping soon".  
- **Timeout**: Inventory reserve timeout = 1s → reject order.  
- **Circuit Breaker**: WMS failures > 50% in 10s → use local inventory snapshot.  
- **Compensation Jobs**:  
  - *Pending order cleaner* (every 5 min): cancel unpaid orders > 30 min → restore inventory.  
  - *Inventory diff reconciler* (hourly): compare service vs WMS → auto fix or alert.  
  - *Stuck order watcher* (every 10 min): mark “Paid” for 2h without shipment → alert.

### Q3. Observability

| Type    | Details                                                | Alert Threshold                         |
|---------|--------------------------------------------------------|-----------------------------------------|
| Logs    | Each state transition with traceId, orderId            | ERROR on reserve/deduct failure         |
| Metrics | Inventory diff ratio, message backlog, stuck orders    | Diff > 1% / backlog > 5000 / stuck > 10 |
| Alerts  | PagerDuty (inventory inconsistency), Slack (slow ops)  | Diff > 100 units, backlog > 1h          |

### Q4. Security

- **Auth**: JWT for all APIs, role based (USER, ADMIN, WMS_CLIENT).  
- **Permission boundaries**: WMS can only call `/shipment`, not `/cancel`.  
- **Sensitive data masking**: Credit card tokenized; logs mask PAN (last 4 only).  
- **Rate limiting**: 100 orders/minute per userId.

### Q5. Recoverability

| Failure Drill                 | Expected Outcome                                            |
|-------------------------------|-------------------------------------------------------------|
| Database failover             | Recovery < 60s, pending orders retried                      |
| MQ broker down                | Orders queued locally, replay after recovery                |
| WMS returns 500 for deduct    | Retry → DLQ → manual repair script available                |
| Negative inventory corruption | Repair API to reset from WMS snapshot, audit log recorded   |

---

## Dedicated Section: Inventory Synchronization Consistency

### Consistency Model Summary

| Operation         | Sync Type           | Consistency Guarantee  | Compensation                      |
|-------------------|---------------------|------------------------|-----------------------------------|
| Reserve           | Strong (DB lock)    | Linearizable           | None                              |
| Deduct (paid)     | Eventual (MQ)       | At least once          | Idempotency + hourly diff checker |
| Cancel restore    | Strong              | Immediate              | Reverse reserve                   |
| Return restore    | Eventual (callback) | At least once          | Reconciliation job                |
| WMS inbound sync  | Batch (15 min)      | Weak                   | Reconciliation job                |

### Test Cases for Consistency

| ID    | Scenario                                   | Steps                                 | Expected                                                 |
|-------|--------------------------------------------|---------------------------------------|----------------------------------------------------------|
| IC-01 | Deduct message lost                        | Simulate MQ ack failure               | Retry until success; after 5 failures → DLQ + alert      |
| IC-02 | Duplicate deduct                           | Send same payment callback twice      | Only one deduct, second returns 200 with idempotent flag |
| IC-03 | Out of order restore (cancel after deduct) | Cancel after shipment started         | Inventory remains sold, refund issued                    |
| IC-04 | WMS diff > threshold                       | Force mismatch                        | Reconciliation job detects, repair order, alerts ops     |
| IC-05 | Concurrent reserve for last item           | 10 requests for qty=1                 | Only one succeeds (pessimistic lock), others get 409     |

### Reconciliation Strategy

- **Hourly job**: Compare `(available+reserved+sold)` local vs WMS snapshot.  
- **Auto fix rules**:  
  - Local < WMS → increment available (WMS source of truth for inbound)  
  - Local > WMS → decrement available, log for fraud check  
- **Manual dashboard** for ops to correct diffs > 100 units.

---

## Exception Handling & Compensation Strategy

### Exception Scenarios

| Exception                            | Detection                                 | Handling                                   | Compensation                          |
|--------------------------------------|-------------------------------------------|--------------------------------------------|---------------------------------------|
| Payment timeout                      | Async callback missing after 15 min       | Cancel order automatically                 | Restore inventory                     |
| Deduct failure after payment         | MQ error handler                          | Retry 3x → DLQ                             | Manual deduct via admin, or refund    |
| WMS shipment fail                    | WMS callback error                        | Order stays Paid, retry shipment           | Re-notify WMS, alert logistics        |
| Partial inventory at shipment        | Warehouse picks                           | Auto‑split shipment + backorder            | Reserve remaining stock, notify user  |
| Return stock restoration fails       | Return completed but inventory not updated| Retry job, log diff                        | Manual inventory adjustment           |

### Compensation Jobs

| Job Name                | Schedule     | Function                                           | Idempotency                     |
|-------------------------|--------------|----------------------------------------------------|---------------------------------|
| PendingOrderCleaner     | Every 5 min  | Cancel unpaid orders >30 min, restore inventory    | Based on order version          |
| DeductRetryHandler      | Every 1 min  | Replay failed deduct messages from DLQ             | Deduct once per orderId         |
| InventoryReconciler     | Every 1 hour | Fix diffs between local and WMS                    | Uses last_sync_time             |
| StuckOrderDetector      | Every 10 min | Find orders “Paid” >2h, no shipment event          | Alert only                      |

### Rollback & Data Repair

- **Admin API**: `POST /admin/repair/inventory` with parameters `{orderId, delta, reason}`. Logs every change.  
- **Disaster recovery**: Restore inventory snapshot + replay transaction log from last 24h.

---

## Release Entry / Exit Criteria

### Entry Criteria (to start testing on staging)

- [ ] All core services (Order, Inventory, Payment, WMS mock) deployed with logs/metrics.  
- [ ] Test data seeded: 100 SKUs (0, low, high stock); 1 flash sale SKU with 10 units.  
- [ ] MQ and reconciliation job infrastructure ready.  
- [ ] Security scan passed (no high-risk vulnerabilities).  
- [ ] Performance baseline: order creation <100ms at 500 rpm.

### Exit Criteria (to release to production)

**Functional**  
- [ ] All P0 test cases passed (100%).  
- [ ] All P1 test cases passed (>95%).  
- [ ] Inventory sync scenarios (IS-01 to IS-08) all pass.  
- [ ] No blocking bug (severity 1 or 2).

**Quality**  
- [ ] Load test: 10,000 concurrent users, error rate <0.1%, P99<300ms for order API.  
- [ ] Chaos test: MQ failure & DB failover recovered within 60s.  
- [ ] Inventory diff ratio <0.01% after 2h of smoke test.  
- [ ] Observability: all critical metrics and alerts verified.

**Business sign-off**  
- [ ] Product owner accepts all happy paths and major exceptions.
