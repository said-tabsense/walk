# Wave 2.1.4: Payment Modes (Organizer Pays vs Individual Pay)

| Field | Value |
|---|---|
| **Feature** | Payment Modes |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Group orders need flexible payment: organizer pays for everyone, or each participant pays their own items.

**Outcome:** Organizer selects mode at checkout. "Organizer pays" creates single order/payment. "Individual pay" splits into per-participant payment requests with 15-minute timeout.

**Metrics:**
- Payment mode selection clear and error-free
- Individual pay requests delivered within 5 seconds
- Zero double-charges
- Partial payment timeout handles correctly at 15 minutes

### 2. User Stories

**US-1: Organizer Pays All**
```
GIVEN organizer closed group order
WHEN they select "I'll pay for everyone" + cash or card
THEN single order created with all items, total charged to organizer, order to POS as one group order.
```

**US-2: Everyone Pays Their Own**
```
GIVEN organizer selects "Everyone pays their own"
WHEN mode confirmed
THEN each participant gets payment request showing their items and total.
  Each completes payment independently (cash or card).
  Order submitted to POS once all paid or after 15-min timeout.
```

**US-3: Partial Payment Timeout**
```
GIVEN 15-minute window active with individual pay
WHEN some participants paid, others haven't after 15 minutes
THEN unpaid items removed; order submitted with paid items only; unpaid participants notified.
```

### 3. API Contracts

**POST /api/v1/marketplace/group-orders/:session_id/checkout**
```
Auth: User JWT (organizer)
Request: { "payment_mode": "organizer_pays | individual_pay", "organizer_payment_method": "cash | card" }
Response 200 (organizer_pays): { "order_id", "total", "payment": { ... } }
Response 200 (individual_pay): { "session_id", "payment_requests": [{ participant_id, items_total, payment_status, payment_deadline }] }
```

**POST /api/v1/marketplace/group-orders/:session_id/pay** (Participant, individual_pay mode)
```
Auth: User JWT (participant)
Request: { "payment_method": "cash | card" }
Response 200: Same as individual payment flow
```

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| payment_mode_title | "Who's paying?" | "من يدفع؟" |
| organizer_pays | "I'll pay for everyone" | "سأدفع عن الجميع" |
| individual_pay | "Everyone pays their own" | "كل شخص يدفع عن نفسه" |
| your_total | "Your total: SAR {total}" | "مجموعك: {total} ر.س" |
| payment_deadline | "Pay within {minutes} minutes" | "ادفع خلال {minutes} دقيقة" |
| payment_expired | "Payment window expired. Your items were removed from the order." | "انتهت مهلة الدفع. تمت إزالة عناصرك من الطلب." |
| all_cancelled | "Group order cancelled. No payments were processed." | "تم إلغاء الطلب الجماعي. لم تتم معالجة أي مدفوعات." |
| mode_locked | "Payment mode cannot be changed after the first payment." | "لا يمكن تغيير طريقة الدفع بعد أول عملية دفع." |

### 5. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | All cash + individual pay | Each marked "cash on delivery"; order submitted immediately |
| E2 | One card fails | Retry within 15-min window; can switch to cash |
| E3 | All timeout | Entire order cancelled |
| E4 | Mode switch after selection | Allowed until first payment received; locked after |

### 6. Wave Dependencies

**Depends On:** Wave 2.1.1 (Session with items); Wave 1.3.3 (Payment integration); Wave 2.1.2 (Checkout UI)
**Exposes To:** Wave 1.3.4 (POS sync receives group order as single order)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

