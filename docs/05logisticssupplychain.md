# Module SRS: Logistics & Supply Chain

Status: Draft v1.0
Parent doc: `docs/MASAR_SRS_v2.0.md` (Section 5)
Build order: 5 of 6

---

## 1. Purpose

Basic inventory tracking and distribution linkage so an organization knows what stock it has and where it went, without needing a full warehouse management system.

## 2. In scope

- Inventory items (name, unit, quantity on hand, warehouse/location).
- Stock movements (receive stock, issue stock to a distribution or department).
- Low-stock threshold alerts.
- Basic stock report (current levels, movement history).

## 3. Out of scope (MVP)

- Multi-warehouse transfer routing/optimization.
- Barcode/RFID scanning integration.
- Supplier/purchase-order management (procurement is a separate preview-only module).

## 4. Data model

```
inventory_items: id, tenant_id, name, unit, quantity_on_hand, low_stock_threshold, location, created_at
stock_movements: id, tenant_id, item_id, type (receive|issue|adjustment), quantity, reference (e.g., distribution_id), moved_by, moved_at
```

## 5. API endpoints

```
GET/POST  /api/inventory-items
PATCH     /api/inventory-items/{id}
POST      /api/stock-movements                 (receive/issue/adjust)
GET       /api/inventory-items?low_stock=true
```

## 6. User stories

- **US-1**: As a Logistics Officer, I want to record incoming stock for an item, so the system reflects what's physically on hand.
- **US-2**: As a Logistics Officer, I want to issue stock against a distribution, so inventory decreases automatically when items go out.
  - AC: Issuing more than `quantity_on_hand` is blocked with a clear error.
- **US-3**: As a Logistics Officer, I want to be alerted when an item drops below its low-stock threshold, so I can reorder in time.
- **US-4**: As a Manager, I want to see stock movement history for an item, so I can audit discrepancies.

## 7. Acceptance criteria (module-level)

- Every stock movement is immutable once recorded (corrections are new `adjustment` entries, not edits), preserving an audit trail.
- Inventory data is tenant-scoped.
- Stock levels update atomically — concurrent issue requests cannot push `quantity_on_hand` negative.
