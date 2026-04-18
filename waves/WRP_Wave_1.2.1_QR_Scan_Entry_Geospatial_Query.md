# Wave 1.2.1: QR Scan Entry + Geospatial Restaurant Query API

| Field | Value |
|---|---|
| **Feature** | QR Scan Entry + Geospatial Restaurant Query API |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** When a user scans an organization's QR code, the system needs to resolve the org ID from the URL, fetch the org's GPS coordinates, and query for all nearby restaurants that have opted into walking delivery. This is the core marketplace discovery engine.

**Outcome:** The QR scan URL (`walk.tabsense.com/m/{org_id}`) resolves to a marketplace page. The backend API returns a sorted list of nearby walking-delivery-enabled restaurants with distance, estimated walk time, operating hours, ratings, and group discount summary.

**Metrics:**
- Geospatial query response < 300ms for up to 100 restaurants in range
- Zero restaurants shown outside configured radius
- Zero restaurants shown with `walking_delivery_enabled = false`
- Distance calculation accurate within 10 meters

**Constraints:**
- PostGIS `ST_DWithin` for geospatial queries
- Only restaurants with `walking_delivery_enabled = true` in `merchant_walk_profiles` appear
- Radius is per-organization (`organizations.walking_radius_m`, default 750m)
- Restaurant data (name, logo, cuisine, rating, hours) fetched from TabSense internal API
- Group discount summary included to incentivize group ordering

### 2. User Stories

**US-1: Resolve QR to Marketplace**

```
GIVEN the user scans a QR code or opens URL "https://walk.tabsense.com/m/{org_id}"
WHEN the frontend loads and calls GET /api/v1/marketplace/:org_id/restaurants
THEN the API validates the org_id exists and is active,
  queries merchant_walk_profiles with ST_DWithin against the org's GPS and walking_radius,
  fetches restaurant details from TabSense internal API for each matching merchant,
  AND returns a sorted list of restaurants with distance, walk time, and metadata.
```

```
GIVEN the org_id does not exist
WHEN the API is called
THEN it returns 404: { "error": "not_found", "message": "Organization not found." }
AND the frontend shows a branded 404 page.
```

```
GIVEN the org exists but status is "inactive" or "deleted"
WHEN the API is called
THEN it returns 410: { "error": "inactive", "message": "This location is no longer active on TabSense Walk." }
AND the frontend shows the branded inactive landing page.
```

**US-2: No Restaurants Nearby**

```
GIVEN the org is active but zero restaurants have walking_delivery_enabled within radius
WHEN the API returns an empty restaurants array
THEN the frontend shows: "No restaurants are available for walking delivery near [Org Name] right now."
  with an option: "Notify me when restaurants join" (email/SMS capture for future use).
```

### 3. API Contracts

**GET /api/v1/marketplace/:org_id/restaurants**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/marketplace/:org_id/restaurants` |
| Auth | None (public) or User JWT (optional, for future personalization) |

**Query Parameters:**
| Param | Type | Default | Description |
|---|---|---|---|
| sort | string | distance | Sort: `distance`, `rating`, `name` |
| page | integer | 1 | Page number |
| limit | integer | 20 | Results per page (max 50) |

**Response 200:**
```json
{
  "organization": {
    "id": "uuid",
    "name": "ACME Corp Tower",
    "name_ar": "برج أكمي",
    "delivery_directions": "Ground floor reception, Building C",
    "walking_radius_m": 750
  },
  "restaurants": [
    {
      "id": "uuid",
      "name": "Shawarma House",
      "name_ar": "بيت الشاورما",
      "logo_url": "https://...",
      "cuisine_type": "Middle Eastern",
      "cuisine_type_ar": "شرق أوسطي",
      "distance_m": 320,
      "estimated_walk_min": 4,
      "rating": 4.5,
      "total_ratings": 42,
      "is_open": true,
      "operating_hours": { "open": "09:00", "close": "23:00" },
      "walking_delivery_notes": "Delivery within 15 min during lunch",
      "group_discount": {
        "enabled": true,
        "min_participants": 5,
        "discount_pct": 10
      }
    }
  ],
  "radius_m": 750,
  "total_count": 8,
  "pagination": { "page": 1, "limit": 20, "total": 8, "pages": 1 }
}
```

| Code | Condition | Body |
|---|---|---|
| 404 | Org not found | `{ "error": "not_found", "message": "Organization not found." }` |
| 410 | Org inactive/deleted | `{ "error": "inactive", "message": "This location is no longer active on TabSense Walk." }` |

**Core Geospatial Query:**
```sql
SELECT
  mwp.tabsense_merchant_id,
  ROUND(ST_Distance(mwp.gps_point, o.gps_point))::INTEGER AS distance_m,
  CEIL(ST_Distance(mwp.gps_point, o.gps_point) / 80)::INTEGER AS estimated_walk_min
FROM merchant_walk_profiles mwp
CROSS JOIN organizations o
WHERE o.id = :org_id
  AND o.status = 'active'
  AND mwp.walking_delivery_enabled = TRUE
  AND ST_DWithin(mwp.gps_point, o.gps_point, o.walking_radius_m)
ORDER BY distance_m ASC
LIMIT :limit OFFSET :offset;
```

Then for each `tabsense_merchant_id`, call TabSense internal API to get restaurant details:
`GET http://tabsense-api.default.svc.cluster.local:8080/internal/merchants/:id`

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| no_restaurants_title | "No Restaurants Available" | "لا توجد مطاعم متاحة" |
| no_restaurants_message | "No restaurants are available for walking delivery near {org_name} right now." | "لا توجد مطاعم متاحة للتوصيل سيراً بالقرب من {org_name} حالياً." |
| notify_me_btn | "Notify me when restaurants join" | "أبلغني عند انضمام مطاعم" |
| distance_label | "{distance}m away" | "على بعد {distance}م" |
| walk_time_label | "~{min} min walk" | "~{min} دقيقة سيراً" |
| closed_label | "Opens at {time}" | "يفتح الساعة {time}" |
| group_discount_badge | "{pct}% off for groups of {min}+" | "خصم {pct}% لمجموعات {min}+" |
| inactive_page | "This location is no longer active on TabSense Walk." | "هذا الموقع لم يعد نشطاً على TabSense Walk." |

### 5. Edge Cases

| # | Scenario | Trigger | Expected Behavior | Response |
|---|---|---|---|---|
| E1 | Org inactive | QR for deactivated org | 410 with branded landing | Inactive page |
| E2 | All restaurants closed | Scan during off-hours | Return restaurants with `is_open: false` | Show grayed with "Opens at [time]" |
| E3 | Restaurant just opted out | Toggle off mid-scan | Excluded from query immediately | Not in results |
| E4 | Dense area (50+ results) | Business district | Paginate; top 20 by distance | Load more button |
| E5 | TabSense internal API slow | > 2s response for merchant data | Cache merchant data for 5 min; serve stale if API slow | Show results from cache with stale indicator |
| E6 | Invalid UUID in path | `/m/not-a-uuid` | 400 | "Invalid organization ID" |

### 6. Wave Dependencies

**Depends On:** Wave 1.1.1 (Org GPS + walking_radius); Wave 1.2.3 (merchant_walk_profiles with GPS and enabled flag); Wave 1.1.3 (QR URL format); TabSense internal API (merchant details)

**Exposes To:** Wave 1.2.2 (Marketplace UI consumes this API); Wave 1.3.1 (Cart uses restaurant data)

**Route Registry:**
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/marketplace/:org_id/restaurants` | GET | Marketplace restaurant list |
| Frontend: `/m/:org_id` | GET | Marketplace entry page |

### 7. QO Testing Guidance

**Risk Areas:** Geospatial accuracy; distance sort order; cache staleness; performance with 50+ restaurants.
**Test Focus:** Boundary testing (restaurant at radius-1m vs radius+1m); distance sort accuracy; verify `is_open` reflects actual operating hours; pagination with varying limits.
**Approach:** Integration-heavy (PostGIS queries); E2E for QR-to-marketplace resolution; performance test with 50+ test merchants.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

