# Wave 2.2.3: Discount Display + Savings Messaging

| Field | Value |
|---|---|
| **Feature** | Discount Display + Savings Messaging |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 1-2 hours |

### 1. Intent Definition

**Problem:** Users need to clearly see discounts and feel good about group ordering. Savings messaging reinforces the value proposition.

**Outcome:** Discount line in cart/checkout. Post-order savings message. Next-tier nudge banner.

### 2. User Stories

**US-1: Savings on Confirmation**
```
GIVEN group order completed with discount
WHEN confirmation page loads
THEN shows: "Your group of {N} saved SAR {X} by ordering together on TabSense Walk!"
```

**US-2: Next Tier Nudge**
```
GIVEN 4 participants, discount starts at 5
WHEN viewing shared cart
THEN banner: "Add 1 more person to unlock 10% group discount!"
```

### 3. Content Specification

| Key | English | Arabic |
|---|---|---|
| savings_message | "Your group of {count} saved SAR {amount} by ordering together!" | "مجموعتك من {count} وفرت {amount} ر.س بالطلب معاً!" |
| nudge_message | "Add {needed} more to unlock {pct}% discount!" | "أضف {needed} آخرين لفتح خصم {pct}%!" |

### 4. Wave Dependencies

**Depends On:** Wave 2.2.2 (Pricing data with discount and next_tier)
**Exposes To:** No downstream; display-only Wave.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

