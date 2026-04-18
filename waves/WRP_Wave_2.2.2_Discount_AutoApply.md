# Wave 2.2.2: Discount Auto-Apply on Group Orders

| Field | Value |
|---|---|
| **Feature** | Discount Auto-Apply |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** When a group order meets a discount threshold, the discount must be automatically calculated and applied at checkout.

**Outcome:** Discount auto-calculates based on participant count vs merchant rules. Highest applicable tier selected. Applied to subtotal. Visible to all participants via real-time update.

**Metrics:**
- Discount calculation < 100ms
- Correct tier applied 100% of the time
- Discount amount accurate to 2 decimal places (halala precision)
- Live update when participant count crosses tier boundary

### 2. User Stories

**US-1: Auto-Apply Discount**
```
GIVEN group order has 7 participants, restaurant rules: 5+ = 10%, 10+ = 15%
WHEN organizer views checkout summary
THEN 10% discount auto-applied. Subtotal shows original, discount line "-SAR X (10% group discount)", total reflects discount.
```

**US-2: Discount Updates Live**
```
GIVEN group order has 4 participants (no discount), 5th joins
WHEN 5th adds items
THEN discount tier activates; all participants see discount appear via WebSocket.
```

**US-3: Participant Leaves**
```
GIVEN 5 participants (10% active), 1 removes items and leaves
WHEN participant count drops to 4
THEN discount removed; all see "Group discount no longer applies."
```

### 3. API Contracts

**GET /api/v1/marketplace/group-orders/:session_id/pricing**
```
Auth: User JWT (any participant)
Response 200: {
  "subtotal": 250.00,
  "discount": { "applied": true, "rule": { "min_participants": 5, "discount_percentage": 10 }, "discount_amount": 25.00, "reason": "10% group discount (5+ participants)" },
  "delivery_fee": 0,
  "total": 225.00,
  "participants_count": 7,
  "next_tier": { "min_participants": 10, "discount_percentage": 15, "participants_needed": 3 } | null
}
```

**Discount Calculation Logic:**
```
1. Get all active discount rules for the merchant, sorted by min_participants DESC
2. Find first rule where session.participants_count >= rule.min_participants
3. discount_amount = MIN(subtotal * (discount_percentage / 100), max_discount_sar || Infinity)
4. total = subtotal - discount_amount + delivery_fee
5. Find next_tier: first rule where min_participants > session.participants_count
```

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| discount_line | "-SAR {amount} ({pct}% group discount)" | "-{amount} ر.س (خصم {pct}% للمجموعة)" |
| discount_removed | "Group discount no longer applies (minimum {min} participants needed)." | "لم يعد خصم المجموعة سارياً (الحد الأدنى {min} مشاركين)." |
| next_tier_nudge | "Add {needed} more to unlock {pct}% discount!" | "أضف {needed} آخرين لفتح خصم {pct}%!" |
| discount_capped | "{pct}% group discount (capped at SAR {cap})" | "خصم {pct}% للمجموعة (بحد أقصى {cap} ر.س)" |

### 5. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | Participant leaves, drops below threshold | Discount removed; totals recalculated |
| E2 | No discount rules | No discount; no discount section in UI |
| E3 | Discount + cap | Capped at max_discount_sar |
| E4 | Near next tier | Show nudge: "Add X more for Y% discount" |

### 6. Wave Dependencies

**Depends On:** Wave 2.2.1 (Discount rules); Wave 2.1.1 (Session participant count)
**Exposes To:** Wave 2.2.3 (Display); Wave 2.1.4 (Payment uses discounted total)
**WebSocket:** Broadcasts pricing update when participant count changes

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

