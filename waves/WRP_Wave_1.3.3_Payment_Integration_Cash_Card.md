# Wave 1.3.3: Payment Integration (Cash + Card)

| Field | Value |
|---|---|
| **Feature** | Payment Integration (Cash + Card) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Orders need payment processing. Cash orders confirm immediately; card orders redirect to Tap/Fatoora gateway.

**Outcome:** Cash orders → `status: confirmed` instantly. Card orders → redirect to payment gateway → webhook callback → `status: confirmed` or `payment_failed`. ZATCA invoice auto-generated on confirmation.

**Metrics:**
- Cash confirmation < 1 second
- Card redirect < 2 seconds
- Payment webhook processing < 5 seconds
- ZATCA invoice auto-generated on 100% of confirmed orders

### 2. User Stories

**US-1: Pay with Cash**
```
GIVEN user selected "Cash" and tapped "Place Order"
WHEN the order is created
THEN status = "confirmed", user sees confirmation page with order number and estimated delivery time,
  restaurant receives order in POS (Wave 1.3.4).
```

**US-2: Pay with Card**
```
GIVEN user selected "Card" and tapped "Place Order"
WHEN the order is created
THEN status = "pending_payment", user redirected to Tap/Fatoora payment page.
  On success: redirect back to confirmation page; status → "confirmed".
  On failure: redirect back to checkout with cart intact.
```

**US-3: Card Payment Fails**
```
GIVEN payment fails or is cancelled
WHEN user returns to Walk
THEN checkout page shows with cart intact: "Payment was not completed. You can try again or choose a different method."
```

### 3. API Contracts

**POST /api/v1/marketplace/orders/:id/pay**
```
Auth: User JWT or Guest session
Request: { "payment_method": "cash | card", "card_provider": "tap | fatoora" }
Response 200 (cash): { "order_id": "uuid", "status": "confirmed", "payment_status": "cash_on_delivery" }
Response 200 (card): { "order_id": "uuid", "status": "pending_payment", "payment_redirect_url": "https://..." }
```

**POST /api/v1/webhooks/payment** (Payment gateway callback)
```
Auth: Gateway signature verification
Side effects: Update order status; trigger POS sync event; generate ZATCA invoice
Response 200: { "received": true }
```

**GET /api/v1/marketplace/orders/:id/status**
```
Auth: User JWT or Guest (phone + order_number)
Response 200: { "order_id", "order_number", "status", "payment_status", "estimated_delivery_min", "zatca_invoice_url" }
```

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| confirmation_title | "Order Confirmed!" | "تم تأكيد الطلب!" |
| order_number_label | "Order Number" | "رقم الطلب" |
| estimated_delivery | "Estimated delivery: ~{min} min" | "وقت التوصيل المتوقع: ~{min} دقيقة" |
| payment_failed | "Payment was not completed. You can try again or choose a different method." | "لم يكتمل الدفع. يمكنك المحاولة مرة أخرى أو اختيار طريقة أخرى." |
| processing_payment | "Processing payment..." | "جاري معالجة الدفع..." |
| cash_note | "Please pay SAR {total} upon delivery." | "يرجى دفع {total} ر.س عند الاستلام." |

### 5. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | Payment webhook delayed > 30s | Poll order status every 5s; timeout at 5 min |
| E2 | Double payment attempt | Idempotency key prevents duplicate |
| E3 | ZATCA invoice generation fails | Order proceeds; invoice queued for retry |
| E4 | Cash order; restaurant rejects | User notified: "Order cancelled. No charge." |

### 6. Wave Dependencies

**Depends On:** Wave 1.3.2 (Order creation); existing Tap/Fatoora integration; ZATCA invoicing
**Exposes To:** Wave 1.3.4 (Confirmed orders trigger POS sync)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

