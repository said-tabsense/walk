# Wave 1.3.4: Order-to-POS Sync + Kitchen Notification

| Field | Value |
|---|---|
| **Feature** | Order-to-POS Sync + Kitchen Notification |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Confirmed marketplace orders need to appear in the restaurant's POS and kitchen display for preparation and walking delivery.

**Outcome:** On order confirmation, an `order.confirmed` event fires through the existing TabSense POS sync pipeline. The order appears in POS as a "TabSense Walk" order with org name, delivery directions, and items. Kitchen display shows order for preparation.

**Metrics:**
- POS sync latency < 10 seconds from confirmation
- 100% order delivery to POS
- Zero data mismatch between marketplace order and POS order

### 2. User Stories

**US-1: Order Appears in POS**
```
GIVEN a marketplace order is confirmed
WHEN the order.confirmed event fires
THEN POS shows: source "TabSense Walk", delivery type "Walking Delivery",
  delivery to: org name + directions + user notes, items with modifiers, payment status.
```

**US-2: Kitchen Display Shows Order**
```
GIVEN order synced to POS
WHEN KDS updates
THEN order appears with "Walking Delivery" tag and items/modifiers.
```

### 3. API Contracts

**Internal Event: `order.confirmed`**
```json
{
  "order_id": "uuid",
  "restaurant_id": "uuid",
  "source": "tabsense_walk",
  "delivery_type": "walking",
  "organization": { "id": "uuid", "name": "string", "delivery_directions": "string", "delivery_notes": "string | null" },
  "items": [ { "menu_item_id": "uuid", "name": "string", "quantity": 2, "modifiers": [...], "price": 18.00 } ],
  "payment": { "method": "cash | card", "status": "paid | cash_on_delivery", "total": 54.00 }
}
```

No new API routes; uses existing TabSense POS sync event pipeline.

### 4. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | POS offline | Queue; retry every 30s for 10 min; alert admin |
| E2 | Menu item mapping fails | Log error; alert ops; cancel order; refund if paid |
| E3 | Duplicate sync event | Idempotent processing (order_id dedup) |

### 5. Wave Dependencies

**Depends On:** Wave 1.3.3 (Payment confirmation); existing POS sync pipeline; existing KDS
**Exposes To:** Surge 3 Wave 3.2.1 (Order status flow)
**New enum values needed:** `order.source = "tabsense_walk"`, `delivery_type = "walking"`

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

