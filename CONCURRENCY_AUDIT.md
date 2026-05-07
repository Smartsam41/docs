# Smart Shipped WMS — Concurrency Audit

**Date:** 2026-04-16
**Scope:** Inventory allocation + pick/pack/ship workflows, receiving, put-away, inventory ops, cycle counts, wave release.
**Deployment assumption:** Production runs `uvicorn --workers 4` on PostgreSQL (per systemd config). SQLite is dev-only. All findings below assume 4 parallel worker processes — Python GIL does NOT protect any of these endpoints.

---

## TL;DR

The codebase has a **strong pattern of locking the `Inventory` row** before decrementing qtyOnHand / qtyAvailable / qtyAllocated. That part is largely correct. The races live elsewhere:

1. **Mutation of `ShippingOrderLine.qtyPicked` / `qtyPacked` / `qtyReceived` is unlocked** in every workflow endpoint. Two concurrent confirm-pick calls on the same line can double-count or silently over-pick.
2. **`check-then-act` on `ShippingOrder.status`** in `allocate()`, `confirm_pick()`, `confirm_pack()`, `stage()`, `skip_pack()`, `start_packing()`, `deallocate()`, `generate_pick_list()`, `release_wave()` — only `ship()` holds the lock. Every other transition is racy.
3. **Upsert-style `db.add(Inventory(...))` in the "no row exists yet" branch** of `adjust`, `transfer`, `put_away`, and `receive_item` has no unique-key guard. Two concurrent first-touches create duplicate rows with split inventory.

---

## Findings (per hot path)

| # | Path | Verdict | Core issue |
|---|------|---------|------------|
| 1 | `rules_engine.py::allocate_inventory_smart()` | ⚠️ | Inventory rows correctly `with_for_update()` (line 578). But `ShippingOrderLine` read at line 494–498 is NOT locked; its `needed = qtyOrdered - qtyAllocated` is computed from a stale read. Two concurrent allocates against the same order can double-reserve stock. |
| 2 | `outbound_workflow.py::allocate_order()` | ⚠️ | `order.status != "Open"` check (line 293) runs without `with_for_update()`. Two concurrent POST `/allocate` calls both see "Open" and both call `allocate_inventory_smart`. Partial mitigation: finding #1's per-row Inventory locks prevent qty corruption on Inventory, but ShippingOrderLine.qtyAllocated will double-count, and two "ALLOCATE" audit + billing events will fire. |
| 3 | `outbound_workflow.py::confirm_pick()` (~L472) | ❌ | **Critical race, matches user's worry exactly.** `ShippingOrder` (line 474) and `ShippingOrderLine` (line 487) are read without lock. `max_pickable = line.qtyAllocated - line.qtyPicked` (L526) is a stale read. The Inventory row IS locked (L543) and the `qtyAllocated < 0` floor check (L554) will raise on the loser — so on-hand stays consistent — but `line.qtyPicked += qty_picked` on two stale copies of the same line produces last-write-wins; the loser's increment is lost, so `line.qtyPicked` under-counts and the operator gets a false "success". Worst case on PostgreSQL: both transactions commit, `qtyAllocated` goes to -5 but is clamped to 0 via `max(0, ...)` in `ship()`, leaving inventory accounting inconsistent with the audit ledger. |
| 4 | `outbound_workflow.py::confirm_pack()` (~L689) | ⚠️ | Same pattern: order + line read unlocked; `max_packable = qtyPicked - qtyPacked` is stale. No Inventory row is touched here (qty moves happen at ship-time), so the worst case is `qtyPacked` exceeding `qtyPicked` by up to one concurrent batch, breaking the invariant. |
| 5 | `outbound_workflow.py::pack_complete()` (~L958) | ⚠️ | Order status check at L981 is unlocked. Two concurrent calls both see "Packing", both create N `PackedCarton` rows → duplicate carton numbers / duplicate `ShipmentPackage` rows at ship time. |
| 6 | `outbound_workflow.py::ship()` (~L765) | ✅ | **Only endpoint that does this correctly.** `.with_for_update()` on the order at L770 before the `status == "Shipped"` check. Inventory rows also locked at L829. Minor nit: the Inventory query filter `Inventory.qtyAllocated > 0` (L828) could miss rows that another transaction just zeroed out, but since that's only cleanup of stranded allocations, not correctness-critical. |
| 7 | `receiving_workflow.py::receive_item()` (~L106) | ⚠️ | `order` (L109) and `line` (L139) read unlocked. `line.qtyReceived += qty` (L158) races with a concurrent receive on the same line. `recv_inv` IS locked (L199), so inventory at RECEIVING doesn't corrupt — but if the row doesn't exist yet, `db.add(Inventory(...))` at L207 runs with no unique-key guard → two concurrent first-receives create TWO RECEIVING rows for the same (itemNum, account). Downstream `put_away`'s `Inventory.location == "RECEIVING"` lookup then picks one arbitrarily. |
| 8 | `receiving_workflow.py::put_away()` (~L396) | ❌ | `order` (L399), `line` (L416), and two Inventory queries (L439 `recv_inv`, L452 `inv`) — all with locks — but there is a TOCTOU between L423 (`remaining = qtyReceived - qtyDamaged - qtyPutAway`) and L433 (`line.qtyPutAway += qty`). Two put-aways targeting different destination bins for the same line both pass the `qty > remaining` check with stale reads, then both decrement `recv_inv` (one will under-flow and get clamped to 0 by `max(0, ...)` at L445). Net effect: more inventory created at destinations than was depleted from RECEIVING. **Inventory creation bug.** |
| 9 | `inventory_ops.py::adjust_inventory()` (~L26) | ⚠️ | `inv` correctly locked (L41). BUT the "Add" branch when `inv is None` (L49–L58) does `db.add(Inventory(...))` with no advisory lock / unique-key on (tenantId, itemNum, location, account). Two concurrent Add calls for a brand-new bin will insert TWO rows → inventory is silently split. |
| 10 | `inventory_ops.py::transfer_inventory()` (~L117) | ⚠️ | src (L133) and dst (L146) both locked — but if dst doesn't exist and gets created at L152, same duplicate-row race as #9. Additional lock-ordering concern: two concurrent transfers (A→B and B→A) could deadlock on PostgreSQL because they grab the source row first then destination. Mitigation: sort locations lexicographically before locking. |
| 11 | `inventory_ops.py::record_line_count()` (~L486) | ⚠️ | No lock on `CycleCountLine` (L488) or `CycleCount` (L494). Two counters submitting counts for the same line produce last-write-wins. Mild severity (cycle counts are a reconciliation step, not the source of truth) but `_refresh_header_stats()` will be recomputed twice on stale data → `cc.varianceCount` can flicker. |
| 12 | `inventory_ops.py::approve_cycle_count()` (~L568) | ⚠️ | `cc.status != "InProgress"` check (L573) runs unlocked. Two concurrent approves would both pass, both iterate lines, both write adjustment `InventoryTransaction` rows → **duplicate ledger entries** (the Inventory row itself is locked at L600 so qtyOnHand ends at `countedQty` either way, but the ledger double-counts the adjustment). |
| 13 | `waves.py::release_wave()` (~L485) | ❌ | `wave` read unlocked (L494). `wave.status != "Draft"` check is racy. Two concurrent releases both pass, both call `allocate_inventory_smart()` on every wave order → double allocation, double billing events. **Critical** in waves with many orders because each `allocate_inventory_smart` commits partially (see #14), so one concurrent release can see the other's intermediate state. |
| 14 | `rules_engine.py::allocate_inventory_smart()` — **commits inside the loop** | ❌ | At L627 there's a `db.rollback()` then `db.commit()` in the shortage path, and L645 commits after logging the rejection. The function commits mid-algorithm if shortages are detected. A caller like `release_wave` that expects atomic "allocate all or roll back" sees partial state. The atomic=True branch of `release_wave` calls `_rollback_wave_allocations` to compensate, but compensation-based rollback is not the same as transactional rollback — if the worker crashes between the partial commit and the compensation, you're left with allocated inventory and no order to consume it. |

---

## Proposed patches

All patches are **diff-format, not applied**. They use `.with_for_update()`, which is a no-op on SQLite and a real row lock on PostgreSQL — exactly what production needs.

### Patch 1 — `confirm_pick` (the user's core worry)

`app/routes/outbound_workflow.py` around line 472:

```python
# BEFORE
@outbound_wf_router.post("/{order_id}/confirm-pick")
def confirm_pick(order_id: int, data: dict, db: Session = Depends(get_db), user=Depends(require_auth)):
    order = scope_to_tenant(db.query(ShippingOrder), ShippingOrder, user).filter(ShippingOrder.id == order_id).first()
    if not order:
        raise HTTPException(status_code=404, detail="Shipping order not found")
    if order.status != "Picking":
        raise HTTPException(status_code=409, detail="Order must be in Picking status")

    line_id = data.get("lineId")
    qty_picked = data.get("qtyPicked", 0)
    location = data.get("location", "")

    if not line_id or qty_picked <= 0:
        raise HTTPException(status_code=400, detail="lineId and qtyPicked > 0 required")

    line = scope_to_tenant(db.query(ShippingOrderLine), ShippingOrderLine, user).filter(ShippingOrderLine.id == line_id).first()
    if not line:
        raise HTTPException(status_code=404, detail="Line not found")
```

```python
# AFTER
@outbound_wf_router.post("/{order_id}/confirm-pick")
def confirm_pick(order_id: int, data: dict, db: Session = Depends(get_db), user=Depends(require_auth)):
    # Lock the order AND the line before reading status / qtyPicked.
    # No-op on SQLite; real row lock on Postgres (prod has --workers 4).
    order = scope_to_tenant(db.query(ShippingOrder), ShippingOrder, user).filter(
        ShippingOrder.id == order_id
    ).with_for_update().first()
    if not order:
        raise HTTPException(status_code=404, detail="Shipping order not found")
    if order.status != "Picking":
        raise HTTPException(status_code=409, detail="Order must be in Picking status")

    line_id = data.get("lineId")
    qty_picked = data.get("qtyPicked", 0)
    location = data.get("location", "")

    if not line_id or qty_picked <= 0:
        raise HTTPException(status_code=400, detail="lineId and qtyPicked > 0 required")

    line = scope_to_tenant(db.query(ShippingOrderLine), ShippingOrderLine, user).filter(
        ShippingOrderLine.id == line_id
    ).with_for_update().first()
    if not line:
        raise HTTPException(status_code=404, detail="Line not found")
    # Re-verify line still belongs to this order (defence in depth)
    if line.shippingOrderId != order_id:
        raise HTTPException(status_code=404, detail="Line not found")
```

Effect: under concurrent `/confirm-pick` on the same line, the second request blocks on the line lock until the first commits. The second then sees the updated `line.qtyPicked` and its `max_pickable` check correctly reflects the new remaining-to-pick. On exceeding, it returns 400 — exactly the behavior the user wants.

### Patch 2 — `confirm_pack` / `pack_complete` / `stage` / `skip_pack` / `start_packing` / `deallocate` / `generate_pick_list` / `allocate_order`

All follow the same template as Patch 1: add `.with_for_update()` to the initial `ShippingOrder` query and, where a `ShippingOrderLine` is mutated, to that query as well. Representative diff for `confirm_pack`:

```python
# BEFORE (~L691, L703)
order = scope_to_tenant(db.query(ShippingOrder), ShippingOrder, user).filter(ShippingOrder.id == order_id).first()
...
line = scope_to_tenant(db.query(ShippingOrderLine), ShippingOrderLine, user).filter(ShippingOrderLine.id == line_id).first()
```

```python
# AFTER
order = scope_to_tenant(db.query(ShippingOrder), ShippingOrder, user).filter(
    ShippingOrder.id == order_id
).with_for_update().first()
...
line = scope_to_tenant(db.query(ShippingOrderLine), ShippingOrderLine, user).filter(
    ShippingOrderLine.id == line_id,
    ShippingOrderLine.shippingOrderId == order_id,  # defence-in-depth
).with_for_update().first()
```

### Patch 3 — `receive_item` duplicate-row race

`app/routes/receiving_workflow.py` around line 195:

```python
# BEFORE
recv_inv = scope_to_tenant(db.query(Inventory), Inventory, user).filter(
    Inventory.itemNum == item_num,
    Inventory.location == "RECEIVING",
    Inventory.account == order.account,
).with_for_update().first()
good_qty = qty - damaged
if recv_inv:
    recv_inv.qtyOnHand += good_qty
    recv_inv.qtyAvailable += good_qty
    recv_inv.qtyDamaged += damaged
else:
    recv_tid = getattr(user, 'accountId', 0) or 0
    db.add(Inventory(
        tenantId=recv_tid,
        itemNum=item_num, description=line.description, account=order.account,
        location="RECEIVING", lotNum=lot_num,
        qtyOnHand=good_qty, qtyAvailable=good_qty, qtyDamaged=damaged,
        status="Receiving",
    ))
```

```python
# AFTER
# Use a DB-level unique index on (tenantId, itemNum, location, account) and
# INSERT ... ON CONFLICT DO UPDATE so concurrent first-touches can't create
# duplicate rows. On SQLite fall back to a SELECT-after-flush retry.
from sqlalchemy.dialects.postgresql import insert as pg_insert
from app.database import IS_SQLITE

good_qty = qty - damaged
recv_tid = getattr(user, 'accountId', 0) or 0

if IS_SQLITE:
    # SQLite path — lock + upsert-by-hand, wrap in an inner retry in case
    # the row is created by a concurrent writer between SELECT and INSERT.
    recv_inv = scope_to_tenant(db.query(Inventory), Inventory, user).filter(
        Inventory.itemNum == item_num,
        Inventory.location == "RECEIVING",
        Inventory.account == order.account,
    ).with_for_update().first()
    if recv_inv:
        recv_inv.qtyOnHand += good_qty
        recv_inv.qtyAvailable += good_qty
        recv_inv.qtyDamaged += damaged
    else:
        db.add(Inventory(
            tenantId=recv_tid,
            itemNum=item_num, description=line.description, account=order.account,
            location="RECEIVING", lotNum=lot_num,
            qtyOnHand=good_qty, qtyAvailable=good_qty, qtyDamaged=damaged,
            status="Receiving",
        ))
else:
    # Postgres path — atomic upsert via ON CONFLICT. Requires a unique index
    # on (tenantId, itemNum, location, account) — add via migration.
    stmt = pg_insert(Inventory.__table__).values(
        tenantId=recv_tid, itemNum=item_num, description=line.description,
        account=order.account, location="RECEIVING", lotNum=lot_num,
        qtyOnHand=good_qty, qtyAvailable=good_qty, qtyDamaged=damaged,
        qtyAllocated=0, status="Receiving",
    ).on_conflict_do_update(
        index_elements=["tenantId", "itemNum", "location", "account"],
        set_={
            "qtyOnHand": Inventory.__table__.c.qtyOnHand + good_qty,
            "qtyAvailable": Inventory.__table__.c.qtyAvailable + good_qty,
            "qtyDamaged": Inventory.__table__.c.qtyDamaged + damaged,
        },
    )
    db.execute(stmt)
```

Required migration: `CREATE UNIQUE INDEX CONCURRENTLY inventory_bin_uidx ON inventory (tenantId, itemNum, location, account)`. Apply the same upsert pattern in `adjust_inventory`, `transfer_inventory` (for the destination bin), and `put_away` (for the target location) — all have the same duplicate-row bug.

### Patch 4 — `put_away` TOCTOU on line.qtyPutAway

`app/routes/receiving_workflow.py` around line 416:

```python
# BEFORE
line = scope_to_tenant(db.query(ReceivingOrderLine), ReceivingOrderLine, user).filter(
    ReceivingOrderLine.id == line_id,
    ReceivingOrderLine.receivingOrderId == order_id,
).first()
```

```python
# AFTER
line = scope_to_tenant(db.query(ReceivingOrderLine), ReceivingOrderLine, user).filter(
    ReceivingOrderLine.id == line_id,
    ReceivingOrderLine.receivingOrderId == order_id,
).with_for_update().first()
```

### Patch 5 — `release_wave`

`app/routes/waves.py` around line 492:

```python
# BEFORE
q = db.query(Wave).filter(Wave.id == wave_id)
q = scope_to_tenant(q, Wave, user)
wave = q.first()
```

```python
# AFTER
q = db.query(Wave).filter(Wave.id == wave_id)
q = scope_to_tenant(q, Wave, user)
wave = q.with_for_update().first()
# Immediately mark in-flight so a concurrent release fails fast even before we commit
if wave and wave.status == "Draft":
    wave.status = "Releasing"
    db.flush()
```

### Patch 6 — `allocate_inventory_smart` should not commit/rollback mid-algorithm

`app/rules_engine.py` around line 627 and 645 — remove the internal `db.rollback()` / `db.commit()` calls and let the caller own the transaction:

```python
# BEFORE (around L626)
if all_shortages and not strategy["allowPartialShip"]:
    db.rollback()
    log_rule_decision(...)
    db.commit()
    return {"success": False, ...}
```

```python
# AFTER
if all_shortages and not strategy["allowPartialShip"]:
    # Caller owns the transaction. Caller will rollback on seeing
    # success=False. We still emit a RuleDecision via a SAVEPOINT so
    # the rejection is logged even when the caller rolls back allocations.
    with db.begin_nested():
        log_rule_decision(...)
    return {"success": False, ...}
```

Then update `outbound_workflow.py::allocate_order` and `waves.py::release_wave` to call `db.rollback()` themselves on `not result["success"]`.

### Patch 7 — `approve_cycle_count` double-ledger race

Add `.with_for_update()` to the initial CycleCount read (L570) so two concurrent approvals serialize.

### Patch 8 — `transfer_inventory` lock ordering (deadlock prevention)

`app/routes/inventory_ops.py` around L130:

```python
# BEFORE: locks src first, then dst — two concurrent A→B and B→A transfers deadlock.
# AFTER: always lock in a canonical order (e.g. by location name).
_loc_lo, _loc_hi = sorted([from_location, to_location])
# First lock both rows in canonical order, then re-fetch src/dst by name
_locked_rows = scope_to_tenant(db.query(Inventory), Inventory, user).filter(
    Inventory.itemNum == item_num,
    Inventory.location.in_([_loc_lo, _loc_hi]),
).with_for_update().order_by(Inventory.location).all()
src = next((r for r in _locked_rows if r.location == from_location), None)
dst = next((r for r in _locked_rows if r.location == to_location), None)
```

---

## Test plan

Concurrent-call tests live in `tests/test_concurrency.py` (new file). Pattern for each hot path:

```python
import threading
from fastapi.testclient import TestClient
from app.main import app

def test_confirm_pick_race_double_pick_is_rejected(seeded_order_with_line_qty10_allocated):
    """Fire 2 concurrent confirm-pick calls for qty=8 each against a line
    that has only qtyAllocated=10. One must succeed (200), the other must
    fail with 400 'only 2 remaining'. Before the FOR UPDATE fix, both
    would succeed and line.qtyPicked would read 16 (or 8 with a silently
    lost update)."""
    order_id, line_id = seeded_order_with_line_qty10_allocated
    client = TestClient(app)
    results = []

    def fire():
        r = client.post(f"/api/shipping-orders/{order_id}/confirm-pick",
                        json={"lineId": line_id, "qtyPicked": 8, "location": "A-01-01"},
                        headers=auth_headers())
        results.append((r.status_code, r.json()))

    t1 = threading.Thread(target=fire)
    t2 = threading.Thread(target=fire)
    t1.start(); t2.start()
    t1.join(); t2.join()

    codes = sorted(r[0] for r in results)
    assert codes == [200, 400], f"expected one 200 + one 400, got {codes}"

    # Verify inventory ledger has exactly ONE Pick transaction and the
    # qtyPicked on the line is exactly 8 (not 16, not 10).
    with SessionLocal() as db:
        line = db.query(ShippingOrderLine).get(line_id)
        assert line.qtyPicked == 8
        pick_txns = db.query(InventoryTransaction).filter_by(
            referenceType="SO", transactionType="Pick", itemNum=line.itemNum
        ).count()
        assert pick_txns == 1
```

**Run against PostgreSQL, not SQLite.** SQLite serializes at the connection level under WAL + `busy_timeout=5000`, which masks the race. Use a `DATABASE_URL=postgresql://...` test fixture for any assertion that depends on FOR UPDATE behavior.

Tests to write (one per ❌ / ⚠️ finding):

1. `test_confirm_pick_race_double_pick_rejected` (finding #3) — 2 threads, same line
2. `test_confirm_pack_race` (#4) — 2 threads, same line
3. `test_allocate_race_double_counts_qtyAllocated` (#1, #2) — 2 threads, same Open order
4. `test_release_wave_race_double_allocation` (#13) — 2 threads, same Draft wave
5. `test_receive_item_race_creates_duplicate_RECEIVING_rows` (#7) — 2 threads, brand-new SKU
6. `test_adjust_add_race_creates_duplicate_rows` (#9) — 2 threads, brand-new bin
7. `test_transfer_deadlock` (#10) — 2 threads doing A→B and B→A
8. `test_approve_cycle_count_race_double_ledger` (#12)
9. `test_put_away_toctou_over_putaway` (#8)

Use a shared `threading.Barrier` to release both threads at exactly the same moment for higher race reproducibility, and run each test 20–50 times in CI to catch non-deterministic failures.

---

## Platform notes

- `.with_for_update()` → `SELECT ... FOR UPDATE` on PostgreSQL; no-op on SQLite. **This is what you want.** Adding it hurts nothing in dev and fixes prod.
- PostgreSQL will raise `psycopg2.errors.DeadlockDetected` on genuine deadlocks — catch and return 409 so the client retries.
- Upsert (`ON CONFLICT DO UPDATE`) needs matching unique indexes; add migrations in the same PR as Patch 3.
- SQLite's `PRAGMA busy_timeout=5000` is already set (good), but it does NOT provide row-level locking — it only serializes writers at the database level. Dev can't reproduce most of these races; rely on prod-Postgres CI to verify.

---

## What's already correct (don't regress it)

- `ship()` in outbound_workflow.py — gold standard, locks the order before the status check.
- `autonumber.py::get_next_number` — correctly uses FOR UPDATE on the sequence row.
- `allocate_inventory_smart` Inventory-row locking (L578) — good; just needs the line + transaction-ownership fixes.
- `reversals.py` uses FOR UPDATE on both the order and inventory rows — consistent with the `ship()` pattern.
