# Wave 1.1.4: Organization Onboarding (Sales + Referral Paths)

| Field | Value |
|---|---|
| **Feature** | Organization Onboarding (Sales + Referral Paths) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Self-registration (Wave 1.1.2) covers one onboarding channel. The sales team needs to create organizations directly (bypassing the review queue), and restaurant owners need a way to refer nearby organizations. Without these channels, onboarding is bottlenecked to self-service only.

**Outcome:** Sales reps can create organizations instantly with `status: active` (QR generated immediately). Restaurants can submit referrals through their dashboard. Referrals enter the admin review queue alongside self-registrations. All channels are tracked via `onboarding_channel` for attribution analytics.

**Metrics:**
- Sales-created orgs active within 1 minute (including QR generation)
- Referral submission < 2 minutes
- Attribution accuracy 100% (`onboarding_channel` correctly set for every org)
- Zero referrals lost

**Constraints:**
- Sales path skips review queue (trusted employees)
- Referral path goes through review queue (like self-registration)
- Referrals may lack full details (contact info optional)

### 2. User Stories

**US-1: Sales Team Creates Organization**

```
GIVEN the sales rep is authenticated with TabSense Sales/Admin role
WHEN they submit organization details via POST /api/v1/admin/organizations
THEN the organization is created with status "active" immediately,
  onboarding_channel = "sales_team",
  QR code generation triggered (Wave 1.1.3),
  reference_number generated,
  AND the sales rep sees confirmation with QR download available within seconds.
```

**US-2: Restaurant Refers Nearby Organization**

```
GIVEN the restaurant owner is on their TabSense merchant dashboard
WHEN they navigate to "Walking Delivery > Refer an Organization" and submit:
  organization name (required), approximate location description (required),
  contact person name (optional), contact phone (optional), notes (optional)
THEN a referral record is created with status "pending_review",
  linked to the referring merchant (referring_merchant_id),
  AND the restaurant sees: "Thank you! Our team will follow up within 48 hours."
```

**US-3: Admin Reviews Referral**

```
GIVEN the admin opens the referral queue
WHEN they view a referral
THEN they see: referring restaurant name, org name, approximate location, contact info,
  and can either:
  a) "Convert to Organization" (creates org record from referral data, triggers review flow)
  b) "Dismiss" (marks referral as not viable)
```

### 3. API Contracts

**POST /api/v1/admin/organizations** (Sales direct creation)

Same as Wave 1.1.1 POST `/api/v1/organizations` but:
- Status automatically set to `active` (overrides default)
- `onboarding_channel` automatically set to `sales_team`
- QR generation triggered on creation (not on separate approval)

**POST /api/v1/merchants/:merchant_id/referrals**

| Property | Value |
|---|---|
| Method | POST |
| Path | `/api/v1/merchants/:merchant_id/referrals` |
| Auth | Merchant JWT |

**Request:**
```json
{
  "organization_name": "string (required, max 200)",
  "approximate_location_description": "string (required, max 500)",
  "contact_person_name": "string (optional, max 100)",
  "contact_person_phone": "string (optional, +966 format)",
  "notes": "string (optional, max 500)"
}
```

**Response 201:**
```json
{
  "referral_id": "uuid",
  "status": "pending_review",
  "referring_merchant_id": "uuid",
  "message": "Thank you! Our team will follow up within 48 hours."
}
```

**GET /api/v1/admin/referrals?status=pending_review**

| Property | Value |
|---|---|
| Method | GET |
| Auth | TabSense Admin JWT |

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "organization_name": "string",
      "approximate_location_description": "string",
      "contact_person_name": "string | null",
      "contact_person_phone": "string | null",
      "notes": "string | null",
      "referring_merchant": { "id": "uuid", "name": "string" },
      "status": "pending_review",
      "created_at": "ISO8601"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total": 5, "pages": 1 }
}
```

**PUT /api/v1/admin/referrals/:id/convert**

Converts a referral into an organization record. Admin provides GPS coordinates (not in the referral) and confirms details.

```json
Request: {
  "gps_lat": "number (required)",
  "gps_lng": "number (required)",
  "delivery_directions": "string (required)",
  "contact_person_name": "string (required if not in referral)",
  "contact_person_phone": "string (required if not in referral)"
}
Response 201: Organization object with status "pending_review", onboarding_channel = "restaurant_referral", referral linked
```

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| referral_title | "Refer a Nearby Organization" | "أحِل منشأة قريبة" |
| referral_subtitle | "Know an office that would love delivery from you? Let us know!" | "تعرف مكتب يحب التوصيل منك؟ أخبرنا!" |
| org_name_label | "Organization Name" | "اسم المنشأة" |
| location_label | "Approximate Location" | "الموقع التقريبي" |
| location_placeholder | "e.g., The building next to Al Faisaliah Tower" | "مثال: المبنى المجاور لبرج الفيصلية" |
| contact_label | "Contact Person (optional)" | "جهة الاتصال (اختياري)" |
| notes_label | "Additional Notes" | "ملاحظات إضافية" |
| submit_btn | "Submit Referral" | "إرسال الإحالة" |
| success_message | "Thank you! Our team will follow up within 48 hours." | "شكراً! فريقنا سيتابع خلال 48 ساعة." |
| duplicate_warning | "A similar organization may already exist. Please review." | "قد تكون هناك منشأة مشابهة موجودة بالفعل. يرجى المراجعة." |

### 5. Edge Cases

| # | Scenario | Trigger | Expected Behavior | Response |
|---|---|---|---|---|
| E1 | Restaurant refers existing org | Name matches existing org within area | API flags duplicate potential; admin sees warning | Referral created with `potential_duplicate: true` |
| E2 | Sales rep creates duplicate GPS | Same GPS as existing active org | 422 same as Wave 1.1.1 | "Organization already exists at this location" |
| E3 | Referral without contact info | All optional fields empty | Allow submission; admin follows up | Created successfully; admin queue shows "Contact info missing" |
| E4 | Referral conversion with bad GPS | Admin enters GPS outside Riyadh | 422 geo-restriction | "TabSense Walk is currently available in Riyadh only" |

### 6. Wave Dependencies

**Depends On:** Wave 1.1.1 (CRUD API); Wave 1.1.3 (QR generation on sales creation)
**Exposes To:** No direct downstream consumers; onboarding paths feed into the organization pool.
**Database:** Uses existing `organizations` table + new `referrals` table (IDP schema).

**Route Registry:**
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/admin/organizations` | POST | Sales direct creation |
| `/api/v1/merchants/:id/referrals` | POST | Merchant referral submission |
| `/api/v1/admin/referrals` | GET | Admin referral queue |
| `/api/v1/admin/referrals/:id/convert` | PUT | Convert referral to org |

### 7. QO Testing Guidance

**Risk Areas:** Attribution accuracy across channels; duplicate detection spanning all 3 channels.
**Test Focus:** Create org via all 3 channels; verify `onboarding_channel` correctly set; verify referral-to-org linkage after conversion.
**Approach:** Integration tests for each channel; E2E for referral-to-approval flow.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

