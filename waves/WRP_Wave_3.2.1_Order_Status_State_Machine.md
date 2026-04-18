# Wave 3.2.1: Order Status State Machine

| Field | Value |
|---|---|
| **Feature** | Order Status State Machine |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** The order lifecycle needs formal state machine with enforced transitions and audit trail.

**Outcome:** Server-side state machine validation. Each transition timestamped. Invalid transitions rejected with 422. Full audit trail in `order_status_history` table.

**Metrics:**
- Zero invalid status transitions in production
- Full audit trail for every order
- Cancellation allowed from any state (with reason)

### 2. State Machine Definition

```
pending_payment → confirmed (on payment success)
confirmed → preparing (merchant starts)
preparing → ready (food ready)
ready → walking (staff leaves)
walking → delivered (staff confirms)
ANY → cancelled (with reason: customer_cancelled | restaurant_cancelled | payment_failed | timeout)

Invalid transitions (return 422):
- confirmed → delivered (skipping steps)
- walking → preparing (going backwards)
- delivered → any (terminal state)
- cancelled → any (terminal state)
```

### 3. Database

Uses `order_status_history` table (IDP schema): `order_id`, `from_status`, `to_status`, `actor`, `created_at`.

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| invalid_transition | "Cannot change status from {from} to {to}." | "لا يمكن تغيير الحالة من {from} إلى {to}." |
| cancellation_reasons | customer_cancelled, restaurant_cancelled, payment_failed, timeout | (enum values, not displayed) |

### 5. Wave Dependencies

**Depends On:** Wave 3.1.3 (Status update endpoint)
**Exposes To:** Wave 3.2.2 (Push on status change); Wave 3.2.3 (Delivery confirmation)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

