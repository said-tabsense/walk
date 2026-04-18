# Wave 3.2.3: Delivery Confirmation + Rating

| Field | Value |
|---|---|
| **Feature** | Delivery Confirmation + Rating |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Walking deliveries need confirmation and a feedback mechanism.

**Outcome:** Restaurant staff confirms delivery in POS. User receives rating prompt 5 minutes after delivery confirmation. 1-5 star rating with optional comment. Rating feeds into restaurant's marketplace listing.

**Metrics:**
- Delivery confirmation rate > 95%
- Rating submission rate > 30%
- Rating prompt appears exactly 5 minutes after delivery confirmation

### 2. User Stories

**US-1: Confirm Delivery (Restaurant)**
```
GIVEN staff walked order to organization
WHEN they mark "delivered" in POS
THEN status updates; user notified; rating prompt queued (5-min delay).
```

**US-2: Rate Delivery (User)**
```
GIVEN user received order and 5 minutes passed
WHEN rating prompt appears (push notification or in-app banner)
THEN they can rate 1-5 stars, leave optional comment, submit.
  Rating attributed to restaurant's walking delivery score.
```

### 3. API Contracts

**POST /api/v1/marketplace/orders/:id/rate**
```
Auth: User JWT
Request: { "rating": 1-5, "comment": "string (optional, max 500)" }
Response 201: { "order_id", "rating", "comment", "rated_at" }
Errors: 409 (already rated), 404 (order not found or not delivered)
```

**GET /api/v1/merchants/:id/walk-ratings**
```
Auth: Merchant JWT
Query: ?period=30d
Response 200: { "average_rating", "total_ratings", "distribution": { "1": N, ... "5": N }, "recent_comments": [...] }
```

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| rate_prompt_title | "How was your delivery?" | "كيف كان التوصيل؟" |
| rate_prompt_subtitle | "Rate your walking delivery experience" | "قيّم تجربة التوصيل سيراً" |
| comment_placeholder | "Any feedback? (optional)" | "أي ملاحظات؟ (اختياري)" |
| submit_rating | "Submit" | "إرسال" |
| thank_you | "Thanks for your feedback!" | "شكراً لملاحظاتك!" |
| already_rated | "You've already rated this order." | "لقد قمت بتقييم هذا الطلب مسبقاً." |
| skip_btn | "Skip" | "تخطي" |

### 5. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | Double rating attempt | 409: "Already rated" |
| E2 | Rating prompt on cancelled order | No prompt sent |
| E3 | Rating 5 minutes after delivery | Delayed job triggers prompt at exactly 5 min |
| E4 | User dismissed prompt | Can rate later via order history link |

### 6. Wave Dependencies

**Depends On:** Wave 3.2.1 (Delivery confirmation status); Wave 3.2.2 (Push for rating prompt)
**Exposes To:** Marketplace listing rating updates (extends Wave 1.2.1 response)
**Database:** `order_ratings` table (IDP schema)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

