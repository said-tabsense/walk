# Wave 2.2.1: Merchant Discount Configuration

| Field | Value |
|---|---|
| **Feature** | Merchant Discount Configuration |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Merchants need to configure group order discounts to incentivize workplace group ordering.

**Outcome:** CRUD API + dashboard UI for merchant discount rules based on participant count thresholds. Multiple tiers supported (e.g., 5+ = 10%, 10+ = 15%).

**Metrics:**
- Discount CRUD response < 500ms
- Discount rules visible in marketplace within 30 seconds
- Correct tier selection for all participant counts

### 2. User Stories

**US-1: Create Discount Rule**
```
GIVEN merchant is on dashboard under "Walking Delivery > Discounts"
WHEN they create rule: min_participants = 5, discount = 10%
THEN rule saved; applies automatically to qualifying group orders.
```

**US-2: Multiple Tiers**
```
GIVEN merchant creates rules: 5+ = 10%, 10+ = 15%, 20+ = 20%
WHEN a group order has 7 participants
THEN highest applicable tier (10%) auto-applied.
```

### 3. API Contracts

**POST /api/v1/merchants/:id/group-discounts**
```
Auth: Merchant JWT
Request: { "min_participants": 5, "discount_percentage": 10.00, "max_discount_sar": 50.00, "active": true }
Response 201: DiscountRule object
```

**GET /api/v1/merchants/:id/group-discounts**
```
Auth: Merchant JWT
Response 200: { "rules": [DiscountRule], sorted by min_participants asc }
```

**PUT /api/v1/merchants/:id/group-discounts/:rule_id**
**DELETE /api/v1/merchants/:id/group-discounts/:rule_id**

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| discounts_title | "Group Discounts" | "خصومات المجموعات" |
| add_rule_btn | "Add Discount Tier" | "إضافة شريحة خصم" |
| min_participants_label | "Minimum Participants" | "الحد الأدنى للمشاركين" |
| discount_pct_label | "Discount (%)" | "نسبة الخصم (%)" |
| max_cap_label | "Maximum Discount (SAR)" | "الحد الأقصى للخصم (ر.س)" |
| max_cap_placeholder | "Optional, e.g., 50" | "اختياري، مثال: 50" |
| no_rules | "No discounts configured. Add a tier to incentivize group orders." | "لا توجد خصومات. أضف شريحة لتحفيز الطلبات الجماعية." |

### 5. Wave Dependencies

**Depends On:** Wave 1.2.3 (Merchant walk profile)
**Exposes To:** Wave 2.2.2 (Discount auto-apply reads rules)
**Database:** `merchant_group_discounts` table (IDP schema)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

