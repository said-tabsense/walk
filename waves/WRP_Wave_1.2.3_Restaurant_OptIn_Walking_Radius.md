# Wave-Ready Package: Wave 1.2.3

## Restaurant Opt-In + Walking Radius Configuration

| Field | Value |
|---|---|
| **Feature** | Restaurant Opt-In + Walking Radius Configuration |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh (BRDG Studio) |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |
| **Surge** | 1 (Foundation) |
| **Flow** | 1.2 (Marketplace Discovery Experience) |

---

## 1. Intent Definition

### Problem
TabSense has thousands of registered restaurants, but not all want to offer walking delivery to nearby offices. There is no mechanism for a merchant to opt in or out of the Walk marketplace, no way to define how far they are willing to walk for deliveries, and no visibility into which organizations they would serve. Without this, Wave 1.2.1 (Geospatial Query) has no way to filter which restaurants appear in the marketplace, and merchants would be involuntarily enrolled in a delivery channel they may not want.

### Outcome
A `merchant_walk_profiles` table exists in Walk PostgreSQL, storing each merchant's walking delivery preferences. Two new API endpoints allow merchants to configure their walking delivery settings and view nearby organizations within range. The merchant's existing TabSense dashboard gains a "Walking Delivery" settings section where they can toggle walking delivery on/off, set their maximum walking radius, and add delivery notes. When enabled, the merchant becomes discoverable in marketplace queries; when disabled, they disappear within 30 seconds.

### Metrics
- Toggle API response time < 500ms (p95)
- Merchant appears/disappears from marketplace geospatial queries within 30 seconds of toggle change
- Nearby organizations list loads in < 300ms for up to 50 organizations in range
- 100% of merchants without GPS coordinates in TabSense are blocked from enabling walking delivery
- Zero merchants appear in marketplace without explicit opt-in (`walking_delivery_enabled = true`)

### Constraints
- Walking delivery configuration is per-branch for multi-location merchants (each branch has its own `tabsense_merchant_id`)
- Merchant GPS coordinates are read from TabSense Aurora MySQL via internal API; Walk PostgreSQL stores a copy in `merchant_walk_profiles.gps_point`
- Walking radius range: 500m minimum, 1000m maximum, 750m default
- Must integrate into the existing TabSense merchant dashboard (not a separate app); the frontend addition is a new settings section
- The Walk API handles the backend; the TabSense frontend team adds the dashboard UI component that calls these endpoints

---

## 2. User Stories

### US-1: Enable Walking Delivery

**As a** restaurant owner, **I want to** enable walking delivery for my restaurant **so that** my restaurant appears in nearby organization marketplaces and I can receive walking delivery orders.

**Acceptance Criteria:**

```
GIVEN the merchant is authenticated with a valid Merchant JWT
  AND the merchant's restaurant has GPS coordinates in TabSense
WHEN they send PUT /api/v1/merchants/:id/walking-delivery with { "enabled": true }
THEN a merchant_walk_profiles record is created (or updated) with:
  - tabsense_merchant_id = merchant's ID
  - gps_point = copied from TabSense internal API merchant GPS
  - walking_delivery_enabled = true
  - max_walking_radius_m = 750 (default) or provided value
  AND the API returns 200 with the profile including nearby_organizations_count
  AND the merchant is immediately discoverable in ST_DWithin marketplace queries
```

```
GIVEN the merchant is authenticated
  AND the merchant's restaurant has NO GPS coordinates in TabSense
WHEN they send PUT /api/v1/merchants/:id/walking-delivery with { "enabled": true }
THEN the API returns 422 with error:
  { "error": "gps_required", "message": "Please set your restaurant's location in Settings before enabling walking delivery." }
  AND no merchant_walk_profiles record is created
```

```
GIVEN the merchant sends PUT with max_walking_radius_m = 499 or 1001
WHEN the API processes validation
THEN the API returns 400 with field-level error:
  { "error": "validation_error", "fields": { "max_walking_radius_m": "Must be between 500 and 1000 meters" } }
```

### US-2: Disable Walking Delivery

**As a** restaurant owner, **I want to** disable walking delivery **so that** I stop appearing in organization marketplaces when I cannot fulfil walking orders.

**Acceptance Criteria:**

```
GIVEN the merchant has walking delivery enabled
WHEN they send PUT /api/v1/merchants/:id/walking-delivery with { "enabled": false }
THEN the merchant_walk_profiles record is updated:
  - walking_delivery_enabled = false
  AND the merchant is excluded from all marketplace ST_DWithin queries
  AND the API returns 200 with nearby_organizations_count = 0
```

```
GIVEN the merchant disables walking delivery while orders are in progress
  (walk_orders with status not in ["delivered", "cancelled"] for this merchant)
WHEN they send PUT with { "enabled": false }
THEN the API returns 200 with a warning:
  { "walking_delivery_enabled": false, "warning": "2 in-progress orders will complete normally. No new marketplace orders will be accepted." }
  AND existing orders are NOT cancelled
```

### US-3: Configure Walking Radius

**As a** restaurant owner, **I want to** set my maximum walking delivery distance **so that** I only receive orders from organizations my staff can reasonably walk to.

**Acceptance Criteria:**

```
GIVEN the merchant has walking delivery enabled
WHEN they send PUT /api/v1/merchants/:id/walking-delivery with { "max_walking_radius_m": 800 }
THEN the profile is updated and the API returns 200 with updated nearby_organizations_count
  (count reflects organizations within the NEW radius, not the old one)
```

```
GIVEN the merchant changes radius from 1000m to 500m
  AND an organization at 700m had been seeing this merchant
WHEN the update completes
THEN that organization's marketplace no longer shows this merchant
  AND the nearby_organizations_count decreases accordingly
```

### US-4: View Nearby Organizations

**As a** restaurant owner, **I want to** see which organizations are within my walking delivery range **so that** I understand my potential customer base and can plan staffing.

**Acceptance Criteria:**

```
GIVEN the merchant has walking delivery enabled with max_walking_radius_m = 750
  AND there are 3 active organizations within 750m of the merchant's GPS
WHEN they send GET /api/v1/merchants/:id/nearby-organizations
THEN the API returns 200 with an array of 3 organizations, each containing:
  id, name, name_ar, distance_m (integer, rounded), estimated_walk_min (integer),
  delivery_directions (first 100 chars as preview)
  AND organizations are sorted by distance_m ascending (closest first)
```

```
GIVEN the merchant has walking delivery enabled
  AND there are no active organizations within range
WHEN they send GET /api/v1/merchants/:id/nearby-organizations
THEN the API returns 200 with an empty array:
  { "organizations": [], "message": "No organizations are registered within your walking range yet." }
```

```
GIVEN the merchant has walking delivery disabled
WHEN they send GET /api/v1/merchants/:id/nearby-organizations
THEN the API returns 200 with an empty array and a note:
  { "organizations": [], "message": "Enable walking delivery to see nearby organizations." }
```

### US-5: View Walking Delivery Settings

**As a** restaurant owner, **I want to** see my current walking delivery configuration **so that** I can review and adjust my settings.

**Acceptance Criteria:**

```
GIVEN the merchant has a merchant_walk_profiles record
WHEN they send GET /api/v1/merchants/:id/walking-delivery
THEN the API returns 200 with:
  { "walking_delivery_enabled": boolean, "max_walking_radius_m": integer,
    "walking_delivery_notes": string|null, "nearby_organizations_count": integer,
    "gps_lat": number, "gps_lng": number, "updated_at": ISO8601 }
```

```
GIVEN the merchant has NO merchant_walk_profiles record (first-time visitor)
WHEN they send GET /api/v1/merchants/:id/walking-delivery
THEN the API returns 200 with defaults:
  { "walking_delivery_enabled": false, "max_walking_radius_m": 750,
    "walking_delivery_notes": null, "nearby_organizations_count": 0,
    "gps_lat": number|null, "gps_lng": number|null, "updated_at": null }
```

---

## 3. API Contracts

### 3.1 GET /api/v1/merchants/:id/walking-delivery

**Get current walking delivery configuration for a merchant.**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/merchants/:id/walking-delivery` |
| Auth | Merchant JWT (merchant can only access their own; TabSense Admin can access any) |
| Content-Type | application/json |

**Path Parameters:**

| Param | Type | Description |
|---|---|---|
| id | UUID | TabSense merchant ID |

**Response 200 (profile exists):**

```json
{
  "merchant_id": "uuid",
  "walking_delivery_enabled": true,
  "max_walking_radius_m": 750,
  "walking_delivery_notes": "Delivery within 15 min during lunch hours",
  "nearby_organizations_count": 5,
  "gps_lat": 24.7677,
  "gps_lng": 46.6384,
  "updated_at": "2026-04-18T10:30:00Z"
}
```

**Response 200 (no profile yet):**

```json
{
  "merchant_id": "uuid",
  "walking_delivery_enabled": false,
  "max_walking_radius_m": 750,
  "walking_delivery_notes": null,
  "nearby_organizations_count": 0,
  "gps_lat": 24.7677,
  "gps_lng": 46.6384,
  "updated_at": null
}
```

Note: `gps_lat` and `gps_lng` are fetched from TabSense internal API even if no walk profile exists, so the dashboard UI can show the merchant's location on a map. If the merchant has no GPS in TabSense, both are `null`.

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid authentication required" }` |
| 403 | Merchant accessing another merchant's profile | `{ "error": "forbidden", "message": "You can only view your own walking delivery settings" }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

### 3.2 PUT /api/v1/merchants/:id/walking-delivery

**Enable, disable, or update walking delivery settings.**

| Property | Value |
|---|---|
| Method | PUT |
| Path | `/api/v1/merchants/:id/walking-delivery` |
| Auth | Merchant JWT (own profile) or TabSense Admin JWT |
| Content-Type | application/json |

**Request Body:**

```json
{
  "enabled": "boolean (required)",
  "max_walking_radius_m": "integer (optional, default: 750, min: 500, max: 1000)",
  "walking_delivery_notes": "string (optional, max 500 chars, e.g. 'Delivery within 15 min')"
}
```

**Response 200 (success):**

```json
{
  "merchant_id": "uuid",
  "walking_delivery_enabled": true,
  "max_walking_radius_m": 750,
  "walking_delivery_notes": "Delivery within 15 min",
  "nearby_organizations_count": 5,
  "gps_lat": 24.7677,
  "gps_lng": 46.6384,
  "updated_at": "2026-04-18T10:30:00Z",
  "warning": null
}
```

**Response 200 (success with warning, when disabling with active orders):**

```json
{
  "merchant_id": "uuid",
  "walking_delivery_enabled": false,
  "max_walking_radius_m": 750,
  "walking_delivery_notes": "Delivery within 15 min",
  "nearby_organizations_count": 0,
  "gps_lat": 24.7677,
  "gps_lng": 46.6384,
  "updated_at": "2026-04-18T10:35:00Z",
  "warning": "2 in-progress orders will complete normally. No new marketplace orders will be accepted."
}
```

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 400 | Validation failure | `{ "error": "validation_error", "fields": { "max_walking_radius_m": "Must be between 500 and 1000 meters" } }` |
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid authentication required" }` |
| 403 | Merchant accessing another's profile | `{ "error": "forbidden", "message": "You can only update your own walking delivery settings" }` |
| 422 | GPS not set in TabSense | `{ "error": "gps_required", "message": "Please set your restaurant's location in Settings before enabling walking delivery." }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

**Backend Logic (for AO agent):**

When `enabled: true` is received:
1. Call TabSense internal API: `GET http://tabsense-api.default.svc.cluster.local:8080/internal/merchants/:id`
2. Extract GPS coordinates from the response
3. If GPS is null/missing, return 422 (gps_required)
4. Upsert into `merchant_walk_profiles`:
   - If record exists: UPDATE `walking_delivery_enabled`, `max_walking_radius_m`, `walking_delivery_notes`, `gps_point`, `updated_at`
   - If record does not exist: INSERT with all fields
5. Count nearby organizations using the geospatial query below
6. Return response

### 3.3 GET /api/v1/merchants/:id/nearby-organizations

**List organizations within the merchant's walking delivery radius.**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/merchants/:id/nearby-organizations` |
| Auth | Merchant JWT (own profile) or TabSense Admin JWT |

**Response 200:**

```json
{
  "organizations": [
    {
      "id": "uuid",
      "name": "ACME Corp Tower",
      "name_ar": "برج شركة أكمي",
      "distance_m": 320,
      "estimated_walk_min": 4,
      "delivery_directions_preview": "Ground floor reception, Building C. Tell security you're from..."
    },
    {
      "id": "uuid",
      "name": "King Abdullah Financial District - Block 3",
      "name_ar": null,
      "distance_m": 580,
      "estimated_walk_min": 7,
      "delivery_directions_preview": "Use the south entrance. Elevator to Floor 2, turn left at..."
    }
  ],
  "message": null
}
```

**Response 200 (no organizations in range):**

```json
{
  "organizations": [],
  "message": "No organizations are registered within your walking range yet."
}
```

**Response 200 (walking delivery disabled):**

```json
{
  "organizations": [],
  "message": "Enable walking delivery to see nearby organizations."
}
```

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid authentication required" }` |
| 403 | Merchant accessing another's data | `{ "error": "forbidden", "message": "You can only view your own nearby organizations" }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

**Geospatial Query (for AO agent):**

```sql
SELECT
  o.id,
  o.name,
  o.name_ar,
  ROUND(ST_Distance(o.gps_point, mwp.gps_point))::INTEGER AS distance_m,
  ROUND(ST_Distance(o.gps_point, mwp.gps_point) / 80)::INTEGER AS estimated_walk_min,
  LEFT(o.delivery_directions, 100) AS delivery_directions_preview
FROM organizations o
CROSS JOIN merchant_walk_profiles mwp
WHERE mwp.tabsense_merchant_id = :merchant_id
  AND mwp.walking_delivery_enabled = TRUE
  AND o.status = 'active'
  AND ST_DWithin(o.gps_point, mwp.gps_point, mwp.max_walking_radius_m)
ORDER BY distance_m ASC;
```

**Walk time estimation:** 80 meters per minute (~4.8 km/h average walking speed). This is a rough estimate; no routing API is used for MVP.

**Nearby organizations count (for PUT response):**

```sql
SELECT COUNT(*)
FROM organizations o
CROSS JOIN merchant_walk_profiles mwp
WHERE mwp.tabsense_merchant_id = :merchant_id
  AND o.status = 'active'
  AND ST_DWithin(o.gps_point, mwp.gps_point, :max_walking_radius_m);
```

---

## 4. Wireframe Descriptions (Agent-Readable)

### Screen: Walking Delivery Settings (Merchant Dashboard)

**Location in app:** Merchant Dashboard → Settings → Walking Delivery (new section)

**Layout:** Vertical card within the existing settings page layout.

**Components (top to bottom):**

1. **Section Header**
   - Title: "Walking Delivery" / "التوصيل سيراً"
   - Subtitle: "Receive orders from nearby offices and deliver by foot" / "استقبل طلبات من المكاتب القريبة ووصّلها سيراً"

2. **Enable Toggle Row**
   - Layout: Horizontal row, full width
   - Left: Label "Enable Walking Delivery" / "تفعيل التوصيل سيراً"
   - Right: Toggle switch (on/off)
   - Height: 48px
   - Background: White card with 1px border, border-radius 8px
   - Padding: 16px horizontal

3. **Configuration Section** (visible only when toggle is ON)
   - **Walking Radius Slider**
     - Label: "Maximum Walking Distance" / "أقصى مسافة للتوصيل سيراً"
     - Slider: range 500-1000, step 50, current value displayed as "{value}m"
     - Default position: 750m (center)
     - Track color: gray-200; filled portion: primary blue (#2563EB)
     - Thumb: 24px circle, white fill, 2px blue border
     - Below slider: min label "500m" (left), max label "1km" (right)

   - **Delivery Notes Input**
     - Label: "Delivery Notes (optional)" / "ملاحظات التوصيل (اختياري)"
     - Placeholder: "e.g., Delivery available 11am-2pm only" / "مثال: التوصيل متاح من 11ص - 2م فقط"
     - Type: textarea, 2 rows, max 500 chars
     - Character counter: bottom-right, "{count}/500"

   - **Save Button**
     - Label: "Save Settings" / "حفظ الإعدادات"
     - Style: Primary blue, full width on mobile, auto width on desktop
     - Height: 44px, border-radius: 8px
     - Loading state: spinner replaces text

4. **Nearby Organizations Card** (visible only when toggle is ON)
   - Header: "Nearby Organizations ({count})" / "المنشآت القريبة ({count})"
   - List items (each):
     - Organization name (bold, 16px)
     - Distance badge: "{distance}m · {walk_min} min walk" in gray-500
     - Delivery directions preview (14px, gray-400, truncated to 1 line)
     - Divider line between items
   - Empty state: Illustration (walking icon) + "No organizations are registered within your walking range yet." centered
   - Max visible: 5, with "View all" link if more

5. **GPS Missing Warning** (visible only when merchant has no GPS)
   - Style: Yellow warning banner, border-radius 8px, padding 12px
   - Icon: Warning triangle (left)
   - Text: "Set your restaurant's location in Settings to enable walking delivery."
   - Action link: "Go to Location Settings →"

**Design Tokens:**

| Token | Value |
|---|---|
| Card background | #FFFFFF |
| Card border | 1px solid #E5E7EB (gray-200) |
| Card border-radius | 12px |
| Card padding | 20px |
| Section gap | 24px between sections |
| Primary action color | #2563EB |
| Toggle track (off) | #D1D5DB (gray-300) |
| Toggle track (on) | #2563EB (primary blue) |
| Toggle thumb | #FFFFFF, 24px diameter |
| Text primary | #111827 (gray-900) |
| Text secondary | #6B7280 (gray-500) |
| Text muted | #9CA3AF (gray-400) |
| Font: section title | 18px, font-weight 600 |
| Font: label | 14px, font-weight 500 |
| Font: body | 14px, font-weight 400 |
| Font: org name in list | 16px, font-weight 600 |
| Warning banner bg | #FEF3C7 (yellow-100) |
| Warning banner border | 1px solid #F59E0B (yellow-500) |
| Warning text | #92400E (yellow-800) |

**UI States:**

| State | Behavior |
|---|---|
| Default (no profile) | Toggle OFF, config section hidden, GPS warning if applicable |
| Enabled | Toggle ON, config section visible, nearby orgs list loaded |
| Loading (toggle change) | Toggle disabled, spinner on save button, "Updating..." text |
| Success (after save) | Green toast: "Walking delivery settings saved" for 3 seconds |
| Error (GPS missing) | Yellow warning banner, toggle click triggers 422 error, banner pulses briefly |
| Error (server) | Red toast: "Something went wrong. Please try again." for 5 seconds |
| Disabled (toggle OFF) | Toggle OFF, config section collapses with animation (200ms ease-out) |

---

## 5. Content Specification

### UI Strings (English)

| Context | Key | English Text |
|---|---|---|
| Section | title | "Walking Delivery" |
| Section | subtitle | "Receive orders from nearby offices and deliver by foot" |
| Toggle | label | "Enable Walking Delivery" |
| Radius | label | "Maximum Walking Distance" |
| Radius | min_label | "500m" |
| Radius | max_label | "1km" |
| Radius | value_display | "{value}m" |
| Notes | label | "Delivery Notes (optional)" |
| Notes | placeholder | "e.g., Delivery available 11am-2pm only" |
| Notes | char_count | "{count}/500" |
| Save | button_label | "Save Settings" |
| Save | loading_label | "Updating..." |
| Nearby | header | "Nearby Organizations ({count})" |
| Nearby | distance_badge | "{distance}m · {walk_min} min walk" |
| Nearby | view_all | "View all" |
| Nearby | empty_title | "No nearby organizations yet" |
| Nearby | empty_description | "No organizations are registered within your walking range yet." |
| Nearby | disabled_message | "Enable walking delivery to see nearby organizations." |
| GPS Warning | message | "Set your restaurant's location in Settings to enable walking delivery." |
| GPS Warning | action | "Go to Location Settings →" |
| Toast | success | "Walking delivery settings saved" |
| Toast | error | "Something went wrong. Please try again." |
| Toast | disabled_success | "Walking delivery disabled" |
| Toast | disabled_warning | "{count} in-progress orders will complete normally. No new marketplace orders will be accepted." |

### UI Strings (Arabic)

| Context | Key | Arabic Text |
|---|---|---|
| Section | title | "التوصيل سيراً" |
| Section | subtitle | "استقبل طلبات من المكاتب القريبة ووصّلها سيراً" |
| Toggle | label | "تفعيل التوصيل سيراً" |
| Radius | label | "أقصى مسافة للتوصيل سيراً" |
| Radius | min_label | "500م" |
| Radius | max_label | "1كم" |
| Radius | value_display | "{value}م" |
| Notes | label | "ملاحظات التوصيل (اختياري)" |
| Notes | placeholder | "مثال: التوصيل متاح من 11ص - 2م فقط" |
| Save | button_label | "حفظ الإعدادات" |
| Save | loading_label | "جاري التحديث..." |
| Nearby | header | "المنشآت القريبة ({count})" |
| Nearby | distance_badge | "{distance}م · {walk_min} دقيقة سيراً" |
| Nearby | view_all | "عرض الكل" |
| Nearby | empty_title | "لا توجد منشآت قريبة بعد" |
| Nearby | empty_description | "لا توجد منشآت مسجلة ضمن نطاق التوصيل الخاص بك حتى الآن." |
| Nearby | disabled_message | "فعّل التوصيل سيراً لعرض المنشآت القريبة." |
| GPS Warning | message | "حدد موقع مطعمك في الإعدادات لتفعيل التوصيل سيراً." |
| GPS Warning | action | "الذهاب لإعدادات الموقع ←" |
| Toast | success | "تم حفظ إعدادات التوصيل سيراً" |
| Toast | error | "حدث خطأ. يرجى المحاولة مرة أخرى." |
| Toast | disabled_success | "تم إيقاف التوصيل سيراً" |
| Toast | disabled_warning | "{count} طلبات قيد التنفيذ ستكتمل بشكل طبيعي. لن يتم قبول طلبات جديدة." |

### API Error Messages (English + Arabic)

| Key | English | Arabic |
|---|---|---|
| gps_required | "Please set your restaurant's location in Settings before enabling walking delivery." | "يرجى تحديد موقع مطعمك في الإعدادات قبل تفعيل التوصيل سيراً." |
| radius_range | "Walking radius must be between 500 and 1000 meters" | "يجب أن يكون نطاق المشي بين 500 و 1000 متر" |
| notes_max_length | "Delivery notes must be 500 characters or fewer" | "يجب ألا تتجاوز ملاحظات التوصيل 500 حرف" |
| unauthenticated | "Valid authentication required" | "يجب تسجيل الدخول" |
| forbidden | "You can only update your own walking delivery settings" | "يمكنك تحديث إعدادات التوصيل الخاصة بك فقط" |
| internal_error | "An unexpected error occurred. Please try again." | "حدث خطأ غير متوقع. يرجى المحاولة مرة أخرى." |

### Character Limits

| Field | Max Length | Notes |
|---|---|---|
| walking_delivery_notes | 500 chars | Optional free-text |
| delivery_directions_preview | 100 chars | Truncated from org.delivery_directions in nearby list |

---

## 6. Database Schema

From IDP Section 4, included for AO agent reference:

```sql
CREATE TABLE merchant_walk_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tabsense_merchant_id UUID NOT NULL UNIQUE,
  gps_point GEOGRAPHY(POINT, 4326) NOT NULL,
  walking_delivery_enabled BOOLEAN DEFAULT FALSE,
  max_walking_radius_m INTEGER DEFAULT 750 CHECK (max_walking_radius_m BETWEEN 500 AND 1000),
  walking_delivery_notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Geospatial index for marketplace queries
CREATE INDEX idx_merchant_walk_gps ON merchant_walk_profiles USING GIST (gps_point);

-- Filter index for enabled merchants
CREATE INDEX idx_merchant_walk_enabled ON merchant_walk_profiles (walking_delivery_enabled)
  WHERE walking_delivery_enabled = TRUE;

-- Unique constraint ensures one profile per merchant branch
-- Already enforced by UNIQUE on tabsense_merchant_id
```

### GPS Sync from TabSense

The merchant's GPS is stored in TabSense Aurora MySQL. On first opt-in (or any update), the Walk API must:

1. Call: `GET http://tabsense-api.default.svc.cluster.local:8080/internal/merchants/:id`
2. Extract `latitude` and `longitude` from response
3. Convert to PostGIS: `ST_SetSRID(ST_MakePoint(longitude, latitude), 4326)::geography`
4. Store in `merchant_walk_profiles.gps_point`

**If the merchant updates their GPS in TabSense later:** The Walk profile's `gps_point` becomes stale. For MVP, this is acceptable. A future enhancement (out of scope) would listen for merchant update events and re-sync GPS.

### Upsert Pattern

```sql
INSERT INTO merchant_walk_profiles (tabsense_merchant_id, gps_point, walking_delivery_enabled, max_walking_radius_m, walking_delivery_notes)
VALUES (:merchant_id, ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography, :enabled, :radius, :notes)
ON CONFLICT (tabsense_merchant_id)
DO UPDATE SET
  gps_point = EXCLUDED.gps_point,
  walking_delivery_enabled = EXCLUDED.walking_delivery_enabled,
  max_walking_radius_m = EXCLUDED.max_walking_radius_m,
  walking_delivery_notes = EXCLUDED.walking_delivery_notes,
  updated_at = NOW();
```

---

## 7. Edge Cases & Error Handling

| # | Scenario | Trigger | Expected Behavior | UI/API Response |
|---|---|---|---|---|
| E1 | GPS not set in TabSense | Enable walking delivery for merchant without GPS | Block enablement | 422: "Please set your restaurant's location in Settings before enabling walking delivery." |
| E2 | No organizations in range | Merchant enables but no orgs within radius | Allow enablement; show empty nearby list | 200: enabled=true, nearby_organizations_count=0, empty org list with message |
| E3 | Disable with active orders | Toggle OFF while walk_orders are in progress | Allow disable; preserve existing orders | 200: enabled=false, warning with active order count |
| E4 | Radius below minimum | max_walking_radius_m = 499 | Block update | 400: "Walking radius must be between 500 and 1000 meters" |
| E5 | Radius above maximum | max_walking_radius_m = 1001 | Block update | 400: same as E4 |
| E6 | Radius non-integer | max_walking_radius_m = 750.5 | Block update | 400: "Walking radius must be a whole number" |
| E7 | Notes too long | walking_delivery_notes > 500 chars | Block update | 400: "Delivery notes must be 500 characters or fewer" |
| E8 | Merchant accesses another's settings | Merchant A sends PUT for Merchant B's ID | Block operation | 403: "You can only update your own walking delivery settings" |
| E9 | TabSense internal API down | Enable request triggers GPS fetch; TabSense API returns 500 | Block enablement; do not create partial profile | 502: "Unable to verify restaurant location. Please try again later." |
| E10 | TabSense internal API returns no merchant | Merchant ID not found in TabSense | Block enablement | 404: "Merchant not found in TabSense" |
| E11 | Concurrent toggle by two users | Two admins toggle same merchant simultaneously | Last-write-wins | 200: whichever completes last persists |
| E12 | Rapid toggle on/off/on | User toggles 3 times in 2 seconds | Debounce on frontend (500ms); API processes each request independently | Frontend only sends last state after debounce |
| E13 | Invalid UUID in path | PUT /api/v1/merchants/not-a-uuid/walking-delivery | Block before processing | 400: "Invalid merchant ID format" |
| E14 | Expired JWT | Request with expired token | Block before processing | 401: "Valid authentication required" |
| E15 | Radius change removes orgs from view | Radius decreased from 1000m to 500m; org at 700m disappears | Immediate effect on marketplace queries | 200: updated nearby_organizations_count reflects new radius |

---

## 8. Wave Dependencies & Integration Map

### Depends On

| Source | What This Wave Consumes | How to Verify |
|---|---|---|
| Wave 1.1.1 (Organization Data Model) | `organizations` table with `gps_point`, `status`, `walking_radius_m` columns. Used in nearby organizations geospatial query. | Run `SELECT id, name, ST_AsText(gps_point::geometry) FROM organizations LIMIT 1;` and confirm data returns. |
| TabSense Merchant Auth System | Merchant JWT issuance and validation. Merchant users must generate JWTs containing `merchant_id` claim. | Authenticate as a test merchant in TabSense and decode the JWT to confirm `merchant_id` is present. |
| TabSense Internal API (Merchant Data) | `GET http://tabsense-api.default.svc.cluster.local:8080/internal/merchants/:id` returns merchant details including GPS (latitude, longitude). | `curl -H "X-Internal-Api-Key: <key>" http://tabsense-api.default.svc.cluster.local:8080/internal/merchants/<test_id>` and confirm GPS fields are present. |
| IDP Prerequisites | Walk PostgreSQL with PostGIS enabled; network connectivity from `walk` namespace to `default` namespace (TabSense API). | Same as Wave 1.1.1 verification. |

### Exposes To

| Downstream Wave | What It Consumes | Contract to Honor |
|---|---|---|
| Wave 1.2.1 (Geospatial Query API) | `merchant_walk_profiles` table; specifically `walking_delivery_enabled = TRUE` filter and `gps_point` column for `ST_DWithin` queries against organization GPS. | Table schema must not change. The `walking_delivery_enabled` column must be BOOLEAN. The `gps_point` column must be `GEOGRAPHY(POINT, 4326)`. The GIST index must exist. |
| Wave 1.2.2 (Marketplace UI) | Marketplace query uses `merchant_walk_profiles` to determine which restaurants to show. Merchant's `walking_delivery_notes` is displayed as a subtitle in the marketplace listing. | `walking_delivery_notes` field must be readable via the marketplace query response. |
| Wave 2.2.1 (Merchant Discount Config) | Extends the merchant walk profile concept; discount configuration lives alongside walking delivery settings. | `merchant_walk_profiles.tabsense_merchant_id` is the key linking Walk features to TabSense merchants. |
| Wave 3.1.2 (Dashboard UI) | Analytics dashboard references `merchant_walk_profiles` to filter data by walking-delivery-enabled merchants. | Same table key. |

### Integration Points

- `merchant_walk_profiles` table lives in Walk PostgreSQL (`walk-postgres.walk.svc.cluster.local:5432`)
- Merchant walking delivery endpoints are under `/api/v1/merchants/:id/walking-delivery` and `/api/v1/merchants/:id/nearby-organizations`, served by `walk-api` pod
- GPS is copied from TabSense Aurora MySQL (via internal API) into Walk PostgreSQL on opt-in; Walk never reads TabSense DB directly
- The merchant dashboard UI change (adding the "Walking Delivery" settings section) requires coordination with the TabSense frontend team. The Walk API provides the backend; the TabSense frontend calls these Walk API endpoints from the existing dashboard
- Service-to-service auth: Walk API calls TabSense internal API with `X-Internal-Api-Key` header

### Route Registry

| Route | Method | File | Purpose |
|---|---|---|---|
| `/api/v1/merchants/:id/walking-delivery` | GET | `routes/api.php` or `routes/merchants.ts` | Get walking delivery config |
| `/api/v1/merchants/:id/walking-delivery` | PUT | Same file | Enable/disable/update walking delivery |
| `/api/v1/merchants/:id/nearby-organizations` | GET | Same file | List nearby organizations |

**Controller:** `MerchantWalkController` (or equivalent)
**Service:** `MerchantWalkService` (business logic: GPS fetch, upsert, nearby org query, active order check)
**Model:** `MerchantWalkProfile` with PostGIS-aware accessors

---

## 9. QO Testing Guidance

### Risk Areas

1. **GPS sync from TabSense:** The internal API call to fetch merchant GPS is a cross-system dependency. If the internal API response schema changes, GPS copy fails silently or stores wrong coordinates.
2. **Toggle propagation latency:** When a merchant toggles walking delivery, the marketplace query results must reflect the change within 30 seconds. If there is caching (CDN, application-level), stale data could show disabled merchants.
3. **Geospatial boundary precision:** The nearby organizations query uses `ST_DWithin` with the merchant's `max_walking_radius_m`. Boundary testing at exactly the radius distance is critical.
4. **Walk time estimation:** The 80m/min assumption is hardcoded. If the AO agent uses a different constant, walk times will be inconsistent.
5. **Authorization matrix:** Merchant JWT can only access their own profile; TabSense Admin can access any. Cross-merchant access must be blocked.

### Suggested Test Focus

- **Integration tests (high priority):**
  - GPS sync roundtrip: call TabSense internal API mock, verify GPS stored correctly in PostGIS, verify lat/lng returned correctly in GET response
  - Toggle on: verify `walking_delivery_enabled = TRUE` in DB; run marketplace-style ST_DWithin query and confirm merchant appears
  - Toggle off: verify merchant disappears from marketplace-style query
  - Nearby organizations query at boundary distances:
    - Organization at 749m with merchant radius 750m: should appear
    - Organization at 750m with merchant radius 750m: should appear (boundary, inclusive)
    - Organization at 751m with merchant radius 750m: should NOT appear
  - Radius change: set radius to 1000m, verify 5 orgs visible; change to 500m, verify only 2 orgs visible (assuming test data)

- **Unit tests (medium priority):**
  - Validation rules: radius range (499, 500, 1000, 1001), notes max length
  - Walk time calculation: 320m = 4 min, 580m = 7 min (verify ROUND(distance/80))
  - Upsert logic: first PUT creates record; second PUT updates existing

- **Authorization tests (medium priority):**
  - Merchant A's JWT accessing Merchant B's endpoints: all 3 endpoints return 403
  - TabSense Admin JWT accessing any merchant: all 3 endpoints return 200
  - Expired JWT: all endpoints return 401
  - Missing JWT: all endpoints return 401

- **Edge case tests:**
  - GPS missing: mock TabSense internal API returning null GPS; verify 422
  - TabSense internal API down: mock 500 response; verify 502 returned to client
  - Disable with active orders: create test walk_orders; verify warning in response; verify orders not cancelled

### Test Approach
- **Primary:** Integration-heavy (cross-system GPS sync + geospatial queries)
- **Secondary:** Unit tests for validation and calculation logic
- **Manual testing:** Toggle walking delivery on a test merchant; open marketplace as employee; verify merchant appears/disappears. Time the propagation.

### Manual Testing Needs
- Verify nearby organizations list: create 3 test organizations at known Riyadh GPS coordinates at varying distances from a test merchant. Confirm distances and walk times are reasonable.
- Verify the dashboard UI renders correctly in both EN and AR (RTL layout check)
- Verify the GPS warning banner appears for a merchant without GPS

### Performance Sensitivity
- PUT endpoint must respond in < 500ms including the TabSense internal API call (allow 200ms for internal call)
- Nearby organizations query must return in < 300ms for up to 50 organizations
- No load testing required at this stage

---

## 10. Sign-Off Log

| Stakeholder | Role | Date | Method | Status |
|---|---|---|---|---|
| Nidal Khalifeh | Product Owner (BRDG Studio) | [PENDING] | Conversation review | [PENDING] |
| TabSense CEO | Business Sponsor | [PENDING] | Email/Meeting | [PENDING] |
| TabSense Frontend Lead | Dashboard UI coordination | [PENDING] | Meeting | [PENDING] |
| SlashTec Lead | Platform Engineer (TabSense internal API) | [PENDING] | IDP review | [PENDING] |

---

## Quality Checklist

### Intent Definition
- [x] Problem statement is specific
- [x] Success metrics are quantifiable (500ms, 30s, 300ms, 100%)
- [x] Target user identified (restaurant owner/merchant)
- [x] Business value articulated (opt-in marketplace participation)
- [x] Constraints listed (per-branch, GPS from TabSense, radius range)

### User Stories
- [x] Each story follows "As a [user], I want [action] so that [benefit]"
- [x] Acceptance criteria use GIVEN/WHEN/THEN and are binary testable
- [x] Edge cases cover errors, empty states, boundaries, authorization, cross-system failures
- [x] No ambiguous language
- [x] Stories scoped to a single Wave

### API Contracts
- [x] All 3 endpoints have HTTP method and path
- [x] Request schemas include all fields with data types
- [x] Response schemas include all fields with data types
- [x] Error responses defined for 400, 401, 403, 422, 500, 502
- [x] Authentication requirements specified per endpoint

### Content Specification
- [x] All UI text is final (zero placeholders)
- [x] All API error messages are final text
- [x] Character limits specified
- [x] Arabic translations included for all UI strings and error messages

### Wireframes
- [x] Dashboard settings screen fully described with component layout
- [x] All UI states covered (default, enabled, loading, success, error, disabled, GPS warning)
- [x] Agent-readable design spec with exact values (colors, sizes, spacing, typography)
- [x] Interactive behaviors documented (toggle, slider, debounce, toast)

### Wave Dependencies
- [x] Depends On lists 4 upstream dependencies with verification commands
- [x] Exposes To lists 4 downstream Waves with specific contracts
- [x] Integration Points provide explicit wiring (service URLs, auth headers, table names)
- [x] Route Registry lists all 3 routes with file paths

### Handoff Readiness
- [x] Error states documented (15 edge cases)
- [ ] Linear ticket created with `wave-ready` label [PENDING]
- [ ] Stakeholder sign-off documented [PENDING]

### QO Readiness
- [x] Acceptance criteria independently testable
- [x] Each GIVEN/WHEN/THEN has single binary result
- [x] Edge cases include specific values (749m, 750m, 751m, 499, 1001)
- [x] QO Testing Guidance complete with risk areas and test focus
- [x] QO receives WRP alongside AO

---

*End of Wave-Ready Package: Wave 1.2.3*
