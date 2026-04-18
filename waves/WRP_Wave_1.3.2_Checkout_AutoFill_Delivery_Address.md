# Wave 1.3.2: Checkout + Auto-Fill Delivery Address

| Field | Value |
|---|---|
| **Feature** | Checkout + Auto-Fill Delivery Address |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Users need to review their order, see the auto-filled delivery address from the organization profile, add personal delivery notes, and submit the order.

**Outcome:** Checkout page showing order summary, auto-filled org address with delivery directions, personal notes field, payment method selection (cash/card), and order submission.

**Metrics:**
- Checkout page load < 1 second
- Address auto-fill 100% accurate from org profile
- Order summary matches cart exactly
- Idempotent order submission (no duplicates on retry)

### 2. User Stories

**US-1: Checkout with Auto-Filled Address**
```
GIVEN the user proceeds to checkout
WHEN the checkout page loads
THEN delivery section shows: org name, delivery directions, access media thumbnails.
  "Personal Delivery Notes" text field for additional instructions.
  User cannot edit org address but can add notes.
```

**US-2: Review Order**
```
GIVEN the user is on checkout
WHEN they review the summary
THEN they see: restaurant name, each item with modifiers and price, subtotal,
  delivery fee (SAR 0), total. Payment method selector: Cash or Card.
  "Place Order" button.
```

**US-3: Guest Checkout**
```
GIVEN the user is not signed in
WHEN they reach checkout for an individual order
THEN additional fields appear: Name (required) and Phone (required, +966 format).
  These are used for delivery contact. No account created.
```

### 3. API Contracts

**POST /api/v1/marketplace/orders**

| Property | Value |
|---|---|
| Method | POST |
| Auth | User JWT or Guest (name + phone in body) |
| Idempotency | `X-Idempotency-Key` header (client-generated UUID) |

**Request:**
```json
{
  "organization_id": "uuid",
  "restaurant_id": "uuid",
  "order_type": "individual",
  "items": [
    {
      "menu_item_id": "uuid",
      "quantity": 2,
      "modifiers": [{ "modifier_id": "uuid", "option_id": "uuid" }],
      "special_instructions": "string (optional, max 200)"
    }
  ],
  "delivery_notes": "string (optional, max 500)",
  "payment_method": "cash | card",
  "customer_name": "string (required for guest)",
  "customer_phone": "string (required for guest, +966 format)"
}
```

**Response 201:**
```json
{
  "order_id": "uuid",
  "order_number": "WALK-20260418-0042",
  "status": "pending_payment | confirmed",
  "items_total": 54.00,
  "discount_amount": 0,
  "delivery_fee": 0,
  "total": 54.00,
  "restaurant": { "name": "string", "estimated_walk_min": 4 },
  "delivery_to": { "organization_name": "string", "delivery_directions": "string", "delivery_notes": "string | null" },
  "payment_url": "string | null"
}
```

| Code | Condition | Body |
|---|---|---|
| 400 | Validation failure | Field-level errors |
| 404 | Restaurant/item not found | `{ "error": "not_found" }` |
| 422 | Item unavailable or price changed | `{ "error": "price_changed", "updated_items": [...] }` |
| 503 | Restaurant closed | `{ "error": "restaurant_closed" }` |

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| checkout_title | "Checkout" | "الدفع" |
| delivery_section | "Delivery To" | "التوصيل إلى" |
| delivery_notes_label | "Personal Delivery Notes (optional)" | "ملاحظات التوصيل الشخصية (اختياري)" |
| delivery_notes_placeholder | "e.g., Floor 3, desk near the window" | "مثال: الطابق 3، المكتب بجانب النافذة" |
| order_summary | "Order Summary" | "ملخص الطلب" |
| subtotal | "Subtotal" | "المجموع الفرعي" |
| delivery_fee | "Delivery Fee" | "رسوم التوصيل" |
| delivery_free | "Free (Walking Delivery)" | "مجاني (توصيل سيراً)" |
| total | "Total" | "الإجمالي" |
| payment_method | "Payment Method" | "طريقة الدفع" |
| cash_option | "Cash on Delivery" | "الدفع عند الاستلام" |
| card_option | "Pay by Card" | "الدفع بالبطاقة" |
| place_order_btn | "Place Order" | "تأكيد الطلب" |
| guest_name_label | "Your Name" | "اسمك" |
| guest_phone_label | "Your Phone" | "رقم هاتفك" |
| restaurant_closed | "Sorry, {restaurant} has closed." | "عذراً، {restaurant} أغلق." |
| price_changed | "Some prices have changed. Please review your updated order." | "تغيرت بعض الأسعار. يرجى مراجعة طلبك المحدث." |
| retry_submit | "Order could not be placed. Tap to retry." | "تعذر تقديم الطلب. انقر للمحاولة مرة أخرى." |

### 5. Edge Cases & Wave Dependencies

Same as condensed WRP in Surge Plan. Key additions: idempotency key handling, guest field validation, org address missing delivery directions fallback.

**Depends On:** Wave 1.3.1 (Cart), Wave 1.2.4 (Auth + org linking), Wave 1.1.1 (Org address)
**Exposes To:** Wave 1.3.3 (Payment), Wave 1.3.4 (POS sync)
**Database:** `walk_orders` table (IDP schema)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

