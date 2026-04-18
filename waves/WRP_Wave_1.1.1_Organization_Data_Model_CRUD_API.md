# Wave-Ready Package: Wave 1.1.1

## Organization Data Model + CRUD API

| Field | Value |
|---|---|
| **Feature** | Organization Data Model + CRUD API |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh (BRDG Studio) |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |
| **Surge** | 1 (Foundation) |
| **Flow** | 1.1 (Organization Profile Management) |

---

## 1. Intent Definition

### Problem
TabSense has no entity representing a physical organization (office building, co-working space, corporate campus). Without this data model, QR codes cannot be linked to delivery locations, geospatial restaurant discovery cannot function, and the entire Walk marketplace has no anchor point. Every downstream feature, from QR scan entry to group ordering, depends on a well-structured organization record with validated GPS coordinates.

### Outcome
A fully functional `organizations` table exists in the Walk PostgreSQL database with PostGIS-enabled geospatial storage. Five RESTful CRUD API endpoints are live, validated, and secured behind TabSense admin authentication. Organizations can be created, listed (with search, filter, pagination), retrieved by ID, updated, and soft-deleted. The data model supports all future Waves: QR generation (1.1.3), geospatial queries (1.2.1), user linking (1.2.4), and self-registration (1.1.2).

### Metrics
- API response time < 200ms for all CRUD operations (p95)
- 100% of CRUD operations enforce validation rules (no invalid records persist)
- Zero orphaned records on soft-delete
- PostGIS `ST_DWithin` index query on `gps_point` returns results in < 50ms for up to 500 organizations
- Duplicate detection (GPS proximity < 50m) correctly blocks with 422 on 100% of attempts

### Constraints
- Must use PostgreSQL 16 with PostGIS extension; column type `GEOGRAPHY(POINT, 4326)` for GPS
- Must integrate with existing TabSense admin JWT authentication for all CRUD endpoints
- Database must be the Walk-specific RDS PostgreSQL instance (NOT Aurora MySQL)
- All API endpoints under `/api/v1/organizations/` path prefix
- Soft-delete pattern only; no hard deletes (status set to "deleted")
- Infrastructure prerequisites (IDP Section 13) must be complete before this Wave starts

---

## 2. User Stories

### US-1: Create Organization Profile

**As a** TabSense admin, **I want to** create an organization profile with name, GPS coordinates, delivery directions, and contact person **so that** a delivery location exists in the system for QR code generation and restaurant discovery.

**Acceptance Criteria:**

```
GIVEN the admin is authenticated with a valid TabSense Admin JWT
  AND the admin sends a POST request to /api/v1/organizations with all required fields
WHEN the API receives the request
THEN the API validates all fields per the schema constraints
  AND creates a new record with id (UUID v4), status "active", 
      created_at and updated_at set to current timestamp
  AND stores GPS coordinates as a PostGIS GEOGRAPHY(POINT, 4326) value
  AND returns 201 with the complete organization object including the generated id
  AND the qr_code_url fields are null (populated by Wave 1.1.3)
```

```
GIVEN the admin sends a POST request with a name exceeding 200 characters
WHEN the API processes validation
THEN the API returns 400 with error body:
  { "error": "validation_error", "fields": { "name": "Must be 200 characters or fewer" } }
```

```
GIVEN the admin sends a POST request with gps_lat outside -90 to 90 range
  OR gps_lng outside -180 to 180 range
WHEN the API processes validation
THEN the API returns 400 with field-level error identifying the invalid coordinate field
```

```
GIVEN the admin sends a POST request with GPS coordinates within 50 meters
  of an existing active organization's gps_point
WHEN the API checks for duplicates using ST_DWithin(existing.gps_point, new.gps_point, 50)
THEN the API returns 422 with error body:
  { "error": "duplicate_location", "message": "An organization already exists near this location",
    "existing_org_id": "<uuid>", "distance_m": <number> }
```

```
GIVEN the admin sends a POST request with walking_radius_m set to 499 or 1001
WHEN the API processes validation
THEN the API returns 400 with field-level error:
  { "error": "validation_error", "fields": { "walking_radius_m": "Must be between 500 and 1000" } }
```

```
GIVEN the admin sends a POST request without a walking_radius_m value
WHEN the API creates the organization
THEN walking_radius_m defaults to 750
```

### US-2: List Organizations (Paginated, Filtered, Searchable)

**As a** TabSense admin, **I want to** view a paginated list of organizations with search and status filtering **so that** I can manage all registered organizations efficiently.

**Acceptance Criteria:**

```
GIVEN the admin is authenticated with a valid TabSense Admin JWT
  AND sends GET /api/v1/organizations without query parameters
WHEN the API processes the request
THEN the API returns 200 with the first 20 organizations (default page=1, limit=20)
  sorted by created_at descending
  AND includes pagination metadata: { page, limit, total, pages }
```

```
GIVEN the admin sends GET /api/v1/organizations?status=active
WHEN the API processes the request
THEN only organizations with status "active" are returned
  AND deleted organizations are never returned regardless of filter
```

```
GIVEN the admin sends GET /api/v1/organizations?search=ACME
WHEN the API processes the request
THEN organizations where name OR name_ar contains "ACME" (case-insensitive) are returned
```

```
GIVEN the admin sends GET /api/v1/organizations?page=999
  AND there are fewer than 999 pages of results
WHEN the API processes the request
THEN the API returns 200 with an empty data array and correct pagination metadata
```

### US-3: Get Organization by ID

**As a** TabSense admin or organization admin, **I want to** retrieve a single organization's full details **so that** I can view or verify its configuration.

**Acceptance Criteria:**

```
GIVEN the user has a valid TabSense Admin JWT OR Org Admin JWT
  AND sends GET /api/v1/organizations/:id with a valid organization UUID
WHEN the API processes the request
THEN the API returns 200 with the complete organization object
```

```
GIVEN the user sends GET /api/v1/organizations/:id with a UUID that does not exist
WHEN the API processes the request
THEN the API returns 404 with error body:
  { "error": "not_found", "message": "Organization not found" }
```

```
GIVEN the user sends GET /api/v1/organizations/:id for a soft-deleted organization
WHEN the API processes the request
THEN the API returns 404 (deleted organizations are not retrievable via this endpoint)
```

### US-4: Update Organization Profile

**As a** TabSense admin or organization admin, **I want to** update an organization's delivery directions, contact info, walking radius, or access media **so that** the information stays current for restaurant delivery staff.

**Acceptance Criteria:**

```
GIVEN the user has a valid TabSense Admin JWT OR Org Admin JWT for this organization
  AND sends PUT /api/v1/organizations/:id with partial fields to update
WHEN the API processes the request
THEN only the provided fields are updated; other fields remain unchanged
  AND updated_at is set to current timestamp
  AND the API returns 200 with the full updated organization object
```

```
GIVEN the user sends a PUT request to update gps_lat or gps_lng
  AND the new coordinates are within 50 meters of another active organization
WHEN the API processes the duplicate check
THEN the API returns 422 with the same duplicate_location error as creation
```

```
GIVEN an Org Admin JWT user attempts to update an organization they do not belong to
WHEN the API checks authorization
THEN the API returns 403 with error body:
  { "error": "forbidden", "message": "You do not have permission to update this organization" }
```

### US-5: Soft-Delete (Deactivate) Organization

**As a** TabSense admin, **I want to** deactivate an organization **so that** its QR code stops working and no new orders can be placed.

**Acceptance Criteria:**

```
GIVEN the admin has a valid TabSense Admin JWT
  AND sends DELETE /api/v1/organizations/:id for an existing active organization
WHEN the API processes the request
THEN the organization's status is set to "deleted"
  AND updated_at is set to current timestamp
  AND the API returns 200 with confirmation body:
    { "message": "Organization deactivated", "id": "<uuid>" }
```

```
GIVEN the organization has in-progress orders (checked via Walk order records with
  status not in ["delivered", "cancelled"])
WHEN the admin sends DELETE
THEN the API returns 200 with a warning in the response:
  { "message": "Organization deactivated", "id": "<uuid>",
    "warning": "3 active orders will complete normally. No new orders will be accepted." }
  AND existing orders are NOT cancelled; they proceed to completion
```

```
GIVEN a non-admin user (Org Admin or unauthenticated) sends DELETE
WHEN the API checks authorization
THEN the API returns 403 (only TabSense admins can delete organizations)
```

---

## 3. API Contracts

### 3.1 POST /api/v1/organizations

**Create a new organization.**

| Property | Value |
|---|---|
| Method | POST |
| Path | `/api/v1/organizations` |
| Auth | TabSense Admin JWT (Bearer token in Authorization header) |
| Content-Type | application/json |

**Request Body:**

```json
{
  "name": "string (required, max 200 chars)",
  "name_ar": "string (optional, max 200 chars)",
  "gps_lat": "number (required, range: -90 to 90, precision: 6 decimal places)",
  "gps_lng": "number (required, range: -180 to 180, precision: 6 decimal places)",
  "delivery_directions": "string (required, max 1000 chars)",
  "delivery_directions_ar": "string (optional, max 1000 chars)",
  "contact_person_name": "string (required, max 100 chars)",
  "contact_person_phone": "string (required, format: +966XXXXXXXXX, 12 chars)",
  "contact_person_email": "string (optional, valid email format, max 200 chars)",
  "walking_radius_m": "integer (optional, default: 750, min: 500, max: 1000)",
  "access_media_urls": "string[] (optional, max 5 items, each valid URL max 2048 chars)",
  "onboarding_channel": "enum: sales_team | restaurant_referral | self_registration (required)"
}
```

**Response 201:**

```json
{
  "id": "uuid",
  "name": "string",
  "name_ar": "string | null",
  "gps_lat": "number",
  "gps_lng": "number",
  "delivery_directions": "string",
  "delivery_directions_ar": "string | null",
  "contact_person_name": "string",
  "contact_person_phone": "string",
  "contact_person_email": "string | null",
  "walking_radius_m": "integer",
  "access_media_urls": "string[]",
  "qr_code_png_url": "null",
  "qr_code_svg_url": "null",
  "status": "active",
  "onboarding_channel": "string",
  "reference_number": "null",
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 400 | Validation failure | `{ "error": "validation_error", "fields": { "<field>": "<message>" } }` |
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid admin authentication required" }` |
| 422 | Duplicate GPS location (< 50m) | `{ "error": "duplicate_location", "message": "An organization already exists near this location", "existing_org_id": "uuid", "distance_m": number }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

### 3.2 GET /api/v1/organizations

**List organizations with pagination, filtering, and search.**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/organizations` |
| Auth | TabSense Admin JWT |

**Query Parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| page | integer | 1 | Page number (min: 1) |
| limit | integer | 20 | Items per page (min: 1, max: 100) |
| status | string | (none) | Filter by status: `active`, `inactive`, `pending_review`, `rejected` |
| search | string | (none) | Case-insensitive search on `name` and `name_ar` |
| sort_by | string | created_at | Sort field: `created_at`, `name`, `status` |
| sort_order | string | desc | Sort direction: `asc`, `desc` |

**Response 200:**

```json
{
  "data": [
    { "...Organization object..." }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 47,
    "pages": 3
  }
}
```

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid admin authentication required" }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

### 3.3 GET /api/v1/organizations/:id

**Get a single organization by ID.**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/organizations/:id` |
| Auth | TabSense Admin JWT OR Org Admin JWT |

**Response 200:** Full organization object (same schema as POST response).

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid authentication required" }` |
| 403 | Org Admin accessing org they don't belong to | `{ "error": "forbidden", "message": "You do not have permission to view this organization" }` |
| 404 | ID not found or deleted | `{ "error": "not_found", "message": "Organization not found" }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

### 3.4 PUT /api/v1/organizations/:id

**Update an organization (partial update).**

| Property | Value |
|---|---|
| Method | PUT |
| Path | `/api/v1/organizations/:id` |
| Auth | TabSense Admin JWT OR Org Admin JWT |
| Content-Type | application/json |

**Request Body:** Any subset of the POST request fields. Only provided fields are updated. `id`, `created_at`, `status`, `qr_code_png_url`, `qr_code_svg_url`, and `reference_number` are read-only and ignored if sent.

**Response 200:** Full updated organization object.

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 400 | Validation failure | Same as POST 400 |
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid authentication required" }` |
| 403 | Org Admin updating org they don't belong to | `{ "error": "forbidden", "message": "You do not have permission to update this organization" }` |
| 404 | ID not found or deleted | `{ "error": "not_found", "message": "Organization not found" }` |
| 422 | Updated GPS creates duplicate | Same as POST 422 |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

### 3.5 DELETE /api/v1/organizations/:id

**Soft-delete (deactivate) an organization.**

| Property | Value |
|---|---|
| Method | DELETE |
| Path | `/api/v1/organizations/:id` |
| Auth | TabSense Admin JWT only (Org Admins cannot delete) |

**Response 200:**

```json
{
  "message": "Organization deactivated",
  "id": "uuid",
  "warning": "string | null"
}
```

The `warning` field is populated only when active orders exist (e.g., `"3 active orders will complete normally. No new orders will be accepted."`).

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 401 | Missing or invalid JWT | `{ "error": "unauthenticated", "message": "Valid admin authentication required" }` |
| 403 | Non-admin user attempts delete | `{ "error": "forbidden", "message": "Only TabSense admins can deactivate organizations" }` |
| 404 | ID not found or already deleted | `{ "error": "not_found", "message": "Organization not found" }` |
| 500 | Server error | `{ "error": "internal_error", "message": "An unexpected error occurred. Please try again." }` |

---

## 4. Wireframe Descriptions (Agent-Readable)

This Wave is primarily backend (data model + API). The admin-facing UI for managing organizations is built across Waves 1.1.2 and 1.1.4. However, this Wave must ensure the API response structure supports the following screens that downstream Waves will render.

### Screen: Organization List (consumed by Wave 1.1.2)

**Purpose:** Admin views all organizations in a paginated table.

**Layout:** Standard admin dashboard table layout.
- **Page Title:** "Organizations" (top-left)
- **Action Button:** "Add Organization" (top-right, primary blue button)
- **Filter Bar:** Status dropdown (All, Active, Inactive, Pending Review, Rejected) + Search text input
- **Table Columns:** Name | Location (city derived from GPS) | Status (badge) | Walking Radius | Created Date | Actions (View, Edit, Delete)
- **Pagination:** Bottom-right, showing "Showing 1-20 of 47" with page navigation

**No UI implementation in this Wave.** This description ensures the API list endpoint returns all data the UI will need.

### Screen: Organization Detail (consumed by Wave 1.1.2)

**Purpose:** Admin views full details of a single organization.

**Layout:** Card-based detail view.
- **Header:** Organization name (large) + status badge + Edit button
- **Section: Location:** Map embed showing GPS pin + delivery directions text
- **Section: Contact:** Person name, phone (clickable), email
- **Section: Configuration:** Walking radius (displayed as "750m"), onboarding channel
- **Section: Access Media:** Grid of uploaded images/videos (thumbnails)
- **Section: QR Code:** Placeholder "QR code not yet generated" (populated by Wave 1.1.3)
- **Section: Metadata:** Created date, last updated, reference number

**No UI implementation in this Wave.** This description ensures the GET by ID endpoint returns all data the detail screen will need.

---

## 5. Content Specification

### API Error Messages (English)

All error messages are returned in the API response body. The frontend (built in later Waves) will display these directly or map them to UI-friendly equivalents.

| Context | Key | English Text |
|---|---|---|
| Validation | name_required | "Organization name is required" |
| Validation | name_max_length | "Organization name must be 200 characters or fewer" |
| Validation | name_ar_max_length | "Arabic name must be 200 characters or fewer" |
| Validation | gps_lat_required | "Latitude is required" |
| Validation | gps_lat_range | "Latitude must be between -90 and 90" |
| Validation | gps_lng_required | "Longitude is required" |
| Validation | gps_lng_range | "Longitude must be between -180 and 180" |
| Validation | delivery_directions_required | "Delivery directions are required" |
| Validation | delivery_directions_max | "Delivery directions must be 1000 characters or fewer" |
| Validation | contact_name_required | "Contact person name is required" |
| Validation | contact_name_max | "Contact person name must be 100 characters or fewer" |
| Validation | contact_phone_required | "Contact phone number is required" |
| Validation | contact_phone_format | "Please enter a valid Saudi phone number (+966XXXXXXXXX)" |
| Validation | contact_email_format | "Please enter a valid email address" |
| Validation | walking_radius_range | "Walking radius must be between 500 and 1000 meters" |
| Validation | access_media_max_count | "Maximum 5 media files allowed" |
| Validation | access_media_url_format | "One or more media URLs are invalid" |
| Validation | onboarding_channel_required | "Onboarding channel is required" |
| Validation | onboarding_channel_invalid | "Onboarding channel must be one of: sales_team, restaurant_referral, self_registration" |
| Duplicate | duplicate_location | "An organization already exists near this location" |
| Auth | unauthenticated | "Valid admin authentication required" |
| Auth | forbidden_view | "You do not have permission to view this organization" |
| Auth | forbidden_update | "You do not have permission to update this organization" |
| Auth | forbidden_delete | "Only TabSense admins can deactivate organizations" |
| Not Found | not_found | "Organization not found" |
| Delete | deactivated | "Organization deactivated" |
| Delete | active_orders_warning | "{count} active orders will complete normally. No new orders will be accepted." |
| Server | internal_error | "An unexpected error occurred. Please try again." |

### API Error Messages (Arabic)

| Context | Key | Arabic Text |
|---|---|---|
| Validation | name_required | "اسم المنشأة مطلوب" |
| Validation | name_max_length | "يجب ألا يتجاوز اسم المنشأة 200 حرف" |
| Validation | gps_lat_required | "خط العرض مطلوب" |
| Validation | gps_lat_range | "يجب أن يكون خط العرض بين -90 و 90" |
| Validation | gps_lng_required | "خط الطول مطلوب" |
| Validation | gps_lng_range | "يجب أن يكون خط الطول بين -180 و 180" |
| Validation | delivery_directions_required | "تعليمات التوصيل مطلوبة" |
| Validation | delivery_directions_max | "يجب ألا تتجاوز تعليمات التوصيل 1000 حرف" |
| Validation | contact_name_required | "اسم جهة الاتصال مطلوب" |
| Validation | contact_phone_required | "رقم هاتف جهة الاتصال مطلوب" |
| Validation | contact_phone_format | "يرجى إدخال رقم هاتف سعودي صالح (+966XXXXXXXXX)" |
| Validation | contact_email_format | "يرجى إدخال بريد إلكتروني صالح" |
| Validation | walking_radius_range | "يجب أن يكون نطاق المشي بين 500 و 1000 متر" |
| Validation | onboarding_channel_required | "قناة التسجيل مطلوبة" |
| Duplicate | duplicate_location | "توجد منشأة أخرى بالقرب من هذا الموقع" |
| Auth | unauthenticated | "يجب تسجيل الدخول كمسؤول" |
| Auth | forbidden_update | "ليس لديك صلاحية لتحديث هذه المنشأة" |
| Auth | forbidden_delete | "فقط مسؤولو TabSense يمكنهم إلغاء تفعيل المنشآت" |
| Not Found | not_found | "المنشأة غير موجودة" |
| Delete | deactivated | "تم إلغاء تفعيل المنشأة" |
| Server | internal_error | "حدث خطأ غير متوقع. يرجى المحاولة مرة أخرى." |

### Character Limits Summary

| Field | Max Length | Notes |
|---|---|---|
| name | 200 | Required |
| name_ar | 200 | Optional |
| delivery_directions | 1000 | Required |
| delivery_directions_ar | 1000 | Optional |
| contact_person_name | 100 | Required |
| contact_person_phone | 12 | Format: +966XXXXXXXXX |
| contact_person_email | 200 | Optional |
| access_media_urls (each) | 2048 | Max 5 URLs |

---

## 6. Database Schema

Directly from the IDP (Section 4), included here for AO agent reference:

```sql
-- Prerequisite: PostGIS extensions must already be enabled (IDP prerequisite #2)
-- CREATE EXTENSION IF NOT EXISTS postgis;
-- CREATE EXTENSION IF NOT EXISTS postgis_topology;

CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  name_ar VARCHAR(200),
  gps_point GEOGRAPHY(POINT, 4326) NOT NULL,
  delivery_directions TEXT NOT NULL,
  delivery_directions_ar TEXT,
  contact_person_name VARCHAR(100) NOT NULL,
  contact_person_phone VARCHAR(20) NOT NULL,
  contact_person_email VARCHAR(200),
  walking_radius_m INTEGER DEFAULT 750 CHECK (walking_radius_m BETWEEN 500 AND 1000),
  access_media_urls TEXT[],
  onboarding_channel VARCHAR(50) NOT NULL,
  status VARCHAR(20) DEFAULT 'pending_review',
  qr_code_png_url TEXT,
  qr_code_svg_url TEXT,
  reference_number VARCHAR(20) UNIQUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Geospatial index for ST_DWithin queries
CREATE INDEX idx_organizations_gps ON organizations USING GIST (gps_point);

-- Status filter index
CREATE INDEX idx_organizations_status ON organizations (status);

-- Name search index (for case-insensitive search)
CREATE INDEX idx_organizations_name_trgm ON organizations USING GIN (name gin_trgm_ops);
CREATE INDEX idx_organizations_name_ar_trgm ON organizations USING GIN (name_ar gin_trgm_ops);
```

**Note on pg_trgm:** The trigram indexes require `CREATE EXTENSION IF NOT EXISTS pg_trgm;` to be run during database setup. Add this to the migration if not already in IDP prerequisites.

### GPS Storage Convention

The API accepts `gps_lat` and `gps_lng` as separate fields for developer convenience. The backend MUST convert these to a PostGIS geography point on insert/update:

```sql
-- Insert: convert lat/lng to PostGIS point
INSERT INTO organizations (name, gps_point, ...)
VALUES ('ACME Corp', ST_SetSRID(ST_MakePoint(lng, lat), 4326)::geography, ...);

-- Read: convert PostGIS point back to lat/lng for API response
SELECT
  id, name,
  ST_Y(gps_point::geometry) AS gps_lat,
  ST_X(gps_point::geometry) AS gps_lng,
  ...
FROM organizations;
```

### Duplicate Detection Query

```sql
-- Check if any active organization exists within 50 meters of the new point
SELECT id, name,
  ST_Distance(gps_point, ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography) AS distance_m
FROM organizations
WHERE status IN ('active', 'pending_review')
  AND ST_DWithin(gps_point, ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)::geography, 50)
LIMIT 1;
```

### Status Enum Values

| Value | Description | Transitions To |
|---|---|---|
| pending_review | Created via self-registration; awaiting admin review | active, rejected |
| active | Operational; QR code works; orders accepted | inactive, deleted |
| inactive | Temporarily paused; QR shows "inactive" page | active, deleted |
| rejected | Admin rejected self-registration submission | (terminal) |
| deleted | Soft-deleted; invisible to all queries except admin audit | (terminal) |

**Note:** Organizations created by TabSense admins via POST start with status "active" (bypassing review). Organizations created via the self-registration endpoint (Wave 1.1.2) start with status "pending_review".

---

## 7. Edge Cases & Error Handling

| # | Scenario | Trigger | Expected Behavior | UI/API Response |
|---|---|---|---|---|
| E1 | Duplicate org at same GPS | POST with GPS within 50m of existing active/pending org | Block creation | 422: "An organization already exists near this location" with existing_org_id and distance_m |
| E2 | Invalid GPS coordinates | gps_lat=91 or gps_lng=181 | Block creation | 400: Field-level validation error on the invalid field |
| E3 | Phone format invalid | contact_person_phone="0512345678" (missing +966) | Block creation | 400: "Please enter a valid Saudi phone number (+966XXXXXXXXX)" |
| E4 | Org with active orders deactivated | DELETE on org with walk_orders in non-terminal status | Allow deactivation; preserve existing orders | 200: Confirmation + warning with active order count |
| E5 | Access media URLs invalid | access_media_urls contains non-URL string | Block creation/update | 400: "One or more media URLs are invalid" |
| E6 | Access media exceeds 5 items | access_media_urls array has 6+ items | Block creation/update | 400: "Maximum 5 media files allowed" |
| E7 | Empty required fields | POST with name="" or missing name | Block creation | 400: Field-level "required" error |
| E8 | Update GPS to duplicate location | PUT with new GPS within 50m of different active org | Block update | 422: Same as E1 |
| E9 | Delete already-deleted org | DELETE on org with status "deleted" | Return not found | 404: "Organization not found" |
| E10 | Org Admin tries to delete | DELETE with Org Admin JWT | Block operation | 403: "Only TabSense admins can deactivate organizations" |
| E11 | Concurrent update conflict | Two admins PUT same org simultaneously | Last-write-wins (no optimistic locking for MVP) | 200: Whichever request completes last persists |
| E12 | Non-UUID in :id parameter | GET /api/v1/organizations/not-a-uuid | Block before query | 400: "Invalid organization ID format" |
| E13 | walking_radius_m non-integer | walking_radius_m=750.5 | Block creation/update | 400: "Walking radius must be a whole number" |
| E14 | Expired JWT | Request with expired token | Block before processing | 401: "Valid admin authentication required" |
| E15 | Search with special characters | search='; DROP TABLE-- | Parameterized query prevents injection | 200: Empty results (no match), no SQL injection |

---

## 8. Wave Dependencies & Integration Map

### Depends On

| Source | What This Wave Consumes | How to Verify |
|---|---|---|
| TabSense Admin Auth System | JWT issuance and validation middleware. Admin users must be able to generate JWTs that this API validates. | Send a request with a valid TabSense admin JWT to any existing TabSense admin endpoint and confirm 200. Verify the JWT payload contains `role: "admin"` or equivalent. |
| IDP Prerequisites (Section 13) | Walk PostgreSQL instance with PostGIS enabled; `walk` namespace in EKS; network connectivity to TabSense internal API | Run `SELECT PostGIS_Version();` against Walk PostgreSQL. Verify the `walk` namespace exists with `kubectl get ns walk`. |

### Exposes To

| Downstream Wave | What It Consumes | Contract to Honor |
|---|---|---|
| Wave 1.1.2 (Self-Registration Portal) | `POST /api/v1/organizations` endpoint to create orgs with `status: pending_review`; `PUT /api/v1/admin/organizations/:id/review` pattern (Wave 1.1.2 adds this endpoint but depends on the organizations table schema) | `organizations` table schema must not change column names or types. The `status` column must accept "pending_review" value. |
| Wave 1.1.3 (QR Code Generation) | `GET /api/v1/organizations/:id` to read org GPS and ID; `PUT /api/v1/organizations/:id` to write `qr_code_png_url` and `qr_code_svg_url` back | The organization object must include `id`, `gps_lat`, `gps_lng`, `name`. The `qr_code_png_url` and `qr_code_svg_url` columns must exist and accept TEXT values via PUT. |
| Wave 1.2.1 (Geospatial Query) | `organizations` table with PostGIS `gps_point` column for `ST_DWithin` queries | `gps_point` column type must be `GEOGRAPHY(POINT, 4326)`. GIST index must exist. Query pattern: `ST_DWithin(orgs.gps_point, merchant.gps_point, org.walking_radius_m)` |
| Wave 1.2.4 (User Linking) | `organizations.id` as foreign key in `user_organizations` table | Organization IDs are UUID v4. The `organizations` table must exist with `id UUID PRIMARY KEY`. |
| Wave 1.3.2 (Checkout Auto-Fill) | `GET /api/v1/organizations/:id` to read `delivery_directions` and `delivery_directions_ar` for auto-fill | These fields must be present in the GET response. |

### Integration Points

- `organizations` table lives in Walk PostgreSQL database (`walk-postgres.walk.svc.cluster.local:5432`)
- All organization CRUD endpoints are under `/api/v1/organizations/` served by `walk-api` pod
- GPS coordinates are stored as PostGIS geography internally but exposed as separate `gps_lat`/`gps_lng` in API responses
- JWT validation calls the existing TabSense auth service at `http://tabsense-api.default.svc.cluster.local:8080/internal/auth/validate`
- No direct database access from other namespaces; all interaction through the Walk API

### Route Registry

| Route | Method | File | Purpose |
|---|---|---|---|
| `/api/v1/organizations` | POST | `routes/api.php` (or `routes/organizations.ts` if Node.js) | Create organization |
| `/api/v1/organizations` | GET | Same file | List organizations |
| `/api/v1/organizations/:id` | GET | Same file | Get single organization |
| `/api/v1/organizations/:id` | PUT | Same file | Update organization |
| `/api/v1/organizations/:id` | DELETE | Same file | Soft-delete organization |

**Controller:** `OrganizationController` (or equivalent)
**Service:** `OrganizationService` (business logic: validation, duplicate detection, GPS conversion)
**Repository/Model:** `Organization` model with PostGIS-aware accessors for `gps_lat`/`gps_lng`

---

## 9. QO Testing Guidance

### Risk Areas

1. **PostGIS geospatial precision:** The 50m duplicate detection boundary is the highest-risk logic. Off-by-one meter at the boundary (49m vs 51m) could cause false positives or missed duplicates.
2. **GPS coordinate conversion:** Converting between `gps_lat/gps_lng` (API) and `GEOGRAPHY(POINT)` (PostGIS) with `ST_MakePoint(lng, lat)` (note: longitude first). Swapping lat/lng is a common bug.
3. **Soft-delete leakage:** Deleted organizations must be invisible in all list/search queries. If the WHERE clause misses a status filter, deleted orgs leak into results.
4. **SQL injection via search:** The `search` query parameter is user-provided text used in a LIKE or trigram query. Must use parameterized queries.
5. **JWT authorization levels:** TabSense Admin vs Org Admin permissions differ per endpoint. The permission matrix must be tested exhaustively.

### Suggested Test Focus

- **Unit tests (high priority):**
  - Validation rules for every field (required, max length, range, format)
  - Phone number format validation (+966XXXXXXXXX pattern)
  - walking_radius_m boundary values: 499 (reject), 500 (accept), 1000 (accept), 1001 (reject)
  - Status transition validation
  - GPS coordinate range validation

- **Integration tests (high priority):**
  - PostGIS point insertion and retrieval roundtrip (verify lat/lng survive conversion)
  - Duplicate detection with ST_DWithin at boundary distances:
    - 49 meters apart: should trigger duplicate (within 50m)
    - 50 meters apart: should trigger duplicate (boundary, inclusive)
    - 51 meters apart: should NOT trigger duplicate
  - Pagination with varying page/limit values
  - Search with partial name matches, Arabic text, special characters
  - Soft-delete: verify GET by ID returns 404, list excludes deleted, direct DB query still shows record

- **Authorization tests (medium priority):**
  - Admin JWT: all 5 endpoints should succeed
  - Org Admin JWT: GET by ID (own org) succeeds, GET by ID (other org) returns 403, DELETE returns 403
  - Expired JWT: all endpoints return 401
  - Missing JWT: all endpoints return 401
  - Malformed JWT: all endpoints return 401

### Test Approach
- **Primary:** Unit-heavy for validation logic + integration-heavy for PostGIS queries
- **Secondary:** API contract tests verifying exact response shapes match this WRP
- **Not needed for this Wave:** E2E tests (no UI in this Wave), load testing (premature at MVP)

### Manual Testing Needs
- Verify GPS accuracy: create an organization with known Riyadh coordinates (e.g., King Abdullah Financial District: 24.7677, 46.6384) and confirm the point renders correctly on a map
- Verify duplicate detection: create two organizations ~45m apart and confirm 422; then ~55m apart and confirm 201

### Performance Sensitivity
- List endpoint with 500+ organizations must return in < 200ms (p95)
- PostGIS GIST index must be verified with `EXPLAIN ANALYZE` on ST_DWithin queries
- No load testing required at this stage

---

## 10. Sign-Off Log

| Stakeholder | Role | Date | Method | Status |
|---|---|---|---|---|
| Nidal Khalifeh | Product Owner (BRDG Studio) | [PENDING] | Conversation review | [PENDING] |
| TabSense CEO | Business Sponsor | [PENDING] | Email/Meeting | [PENDING] |
| SlashTec Lead | Platform Engineer | [PENDING] | IDP review confirmation | [PENDING] |

---

## Quality Checklist

### Intent Definition
- [x] Problem statement is specific (no "improve user experience")
- [x] Success metrics are quantifiable with numbers (200ms, 100%, 50ms, 50m)
- [x] Target user identified by role (TabSense admin, Org admin)
- [x] Business value articulated (foundation for all Walk features)
- [x] Constraints listed (PostGIS, JWT auth, soft-delete, Walk PostgreSQL)

### User Stories
- [x] Each story follows "As a [user], I want [action] so that [benefit]"
- [x] Acceptance criteria use GIVEN/WHEN/THEN and are testable (binary true/false)
- [x] Edge cases cover: errors, empty states, boundaries, authorization
- [x] No ambiguous language ("should," "nice," "appropriate")
- [x] Stories are scoped to a single Wave

### API Contracts
- [x] All 5 endpoints have HTTP method and path
- [x] Request schemas include all required fields with data types
- [x] Response schemas include all fields with data types
- [x] Error responses defined for 400, 401, 403, 404, 422, 500
- [x] Authentication requirements specified per endpoint
- [x] Pagination defined for list endpoint

### Content Specification
- [x] All API error messages are final text (zero placeholders)
- [x] Error messages are actionable and user-friendly
- [x] Character limits specified for all text inputs
- [x] Arabic translations included for all error messages

### Wireframes
- [x] Downstream screen descriptions included for AO agent context
- [x] Navigation flow documented (N/A; backend-only Wave)
- [x] All UI states described for downstream consumption
- [x] Agent-readable design spec included (screen layouts for downstream Waves)

### Wave Dependencies
- [x] Depends On lists upstream dependencies with verification steps
- [x] Exposes To lists all 5 downstream Waves with specific contracts
- [x] Integration Points provide explicit wiring (service URLs, column types)
- [x] Route Registry lists all 5 routes with file paths and methods

### Handoff Readiness
- [x] Error states documented (15 edge cases)
- [ ] Linear ticket created with `wave-ready` label [PENDING]
- [ ] Stakeholder sign-off documented [PENDING]

### QO Readiness
- [x] Acceptance criteria are independently testable by automated suites
- [x] Each GIVEN/WHEN/THEN has a single binary observable result
- [x] Edge cases include specific input values (49m, 50m, 51m, 499, 500, 1000, 1001)
- [x] QO Testing Guidance section complete with risk areas and test focus
- [x] QO receives WRP alongside AO

---

*End of Wave-Ready Package: Wave 1.1.1*
