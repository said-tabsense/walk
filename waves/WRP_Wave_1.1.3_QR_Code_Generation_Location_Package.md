# Wave 1.1.3: QR Code Generation + Location Package

| Field | Value |
|---|---|
| **Feature** | QR Code Generation + Location Package |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Each approved organization needs a unique QR code encoding the marketplace URL with the org's ID. The QR code is the primary physical entry point for the entire TabSense Walk experience. Without it, employees have no way to discover the marketplace.

**Outcome:** QR codes are auto-generated when an organization is approved (status → "active"). Generated as branded PNG (300dpi, A4 print-ready) and SVG. Stored in S3 (`tabsense-walk-qr-assets` bucket) with pre-signed URLs. Organization record updated with `qr_code_png_url` and `qr_code_svg_url`. Org admins can download both formats from their portal.

**Metrics:**
- QR generation < 2 seconds from approval event
- 100% scan success rate on iOS Safari + Android Chrome native cameras
- QR encodes correct URL: `https://walk.tabsense.com/m/{org_id}`
- Print quality: 300dpi, min 3cm x 3cm scannable at arm's length

**Constraints:**
- QR must be scannable by native phone cameras (no app required)
- Must include TabSense Walk branding (logo, "Scan to order food" text)
- Generation runs as background job on `walk-worker` pod
- QR assets stored in `tabsense-walk-qr-assets` S3 bucket

### 2. User Stories

**US-1: Auto-Generate QR on Approval**

```
GIVEN an organization's status changes to "active" (via admin approval or sales creation)
WHEN the approval event fires
THEN the walk-worker pod:
  1. Generates QR code encoding "https://walk.tabsense.com/m/{org_id}"
  2. Renders branded PNG (300dpi, 2480x3508px A4, white bg, TabSense Walk logo top, QR center, org name + "Scan to order food" bottom)
  3. Renders SVG (same layout, scalable)
  4. Uploads both to S3 bucket "tabsense-walk-qr-assets" at key: qr/{org_id}/qr.png and qr/{org_id}/qr.svg
  5. Updates organization record: qr_code_png_url and qr_code_svg_url with S3 pre-signed URLs (7-day expiry, auto-refresh)
```

**US-2: Download QR Assets**

```
GIVEN the organization is active and has a generated QR code
WHEN the org admin sends GET /api/v1/organizations/:id/qr-code
THEN the API returns QR asset URLs (fresh pre-signed URLs) and metadata
  AND the admin can download PNG (for printing) and SVG (for design use)
```

**US-3: Regenerate QR**

```
GIVEN the org admin requests QR regeneration (e.g., old QR damaged/leaked)
WHEN they send POST /api/v1/organizations/:id/qr-code/regenerate
THEN a new QR code is generated with the same URL encoding
  AND old S3 assets are marked deprecated (not deleted; redirect maintained)
  AND the organization record is updated with new URLs
```

### 3. API Contracts

**POST /api/v1/organizations/:id/qr-code** (internal, triggered by approval)

| Property | Value |
|---|---|
| Method | POST |
| Path | `/api/v1/organizations/:id/qr-code` |
| Auth | System/Internal (X-Internal-Api-Key header) |

**Response 201:**
```json
{
  "organization_id": "uuid",
  "qr_code_png_url": "https://tabsense-walk-qr-assets.s3.amazonaws.com/qr/{org_id}/qr.png?...",
  "qr_code_svg_url": "https://tabsense-walk-qr-assets.s3.amazonaws.com/qr/{org_id}/qr.svg?...",
  "encoded_url": "https://walk.tabsense.com/m/{org_id}",
  "generated_at": "ISO8601"
}
```

**GET /api/v1/organizations/:id/qr-code**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/organizations/:id/qr-code` |
| Auth | Org Admin JWT or TabSense Admin JWT |

**Response 200:**
```json
{
  "qr_code_png_url": "string (fresh pre-signed URL)",
  "qr_code_svg_url": "string (fresh pre-signed URL)",
  "encoded_url": "https://walk.tabsense.com/m/{org_id}",
  "generated_at": "ISO8601"
}
```

| Code | Condition | Body |
|---|---|---|
| 404 | Org not found or QR not yet generated | `{ "error": "not_found", "message": "QR code not found. It may still be generating." }` |

**POST /api/v1/organizations/:id/qr-code/regenerate**

| Property | Value |
|---|---|
| Method | POST |
| Auth | Org Admin JWT or TabSense Admin JWT |

**Response 201:** Same as POST qr-code response with new URLs.

### 4. Wireframe Descriptions

**QR Code Poster Layout (PNG output):**
- **Dimensions:** 2480 x 3508px (A4 at 300dpi)
- **Background:** White (#FFFFFF)
- **Top section (10% height):** TabSense Walk logo centered, 200px width
- **Center section (60% height):** QR code, 1500x1500px, centered, with quiet zone margin
- **Bottom section (20% height):**
  - Organization name: 48px, bold, #111827, centered
  - "Scan to order food" / "امسح للطلب": 36px, #6B7280, centered
  - Walking icon (SVG inline): 48px, centered below text
- **Bottom bar (10% height):** "Powered by TabSense" in small text, centered

**QR Download Section (in Org Admin Portal):**
- Card with QR preview (200x200px thumbnail)
- Two download buttons side by side: "Download PNG (Print)" and "Download SVG"
- "Regenerate QR Code" link below (with confirmation dialog)

### 5. Content Specification

| Key | English | Arabic |
|---|---|---|
| poster_cta | "Scan to order food" | "امسح للطلب" |
| poster_powered_by | "Powered by TabSense" | "بتقنية TabSense" |
| download_png_btn | "Download PNG (Print-Ready)" | "تحميل PNG (جاهز للطباعة)" |
| download_svg_btn | "Download SVG" | "تحميل SVG" |
| regenerate_link | "Regenerate QR Code" | "إعادة إنشاء رمز QR" |
| regenerate_confirm | "Generate a new QR code? The old one will still work." | "إنشاء رمز QR جديد؟ الرمز القديم سيظل يعمل." |
| qr_not_generated | "QR code is being generated. Please check back in a moment." | "جاري إنشاء رمز QR. يرجى المحاولة لاحقاً." |
| inactive_landing | "This location is no longer active on TabSense Walk." | "هذا الموقع لم يعد نشطاً على TabSense Walk." |
| coming_soon_landing | "Coming soon! TabSense Walk is launching here soon." | "قريباً! TabSense Walk سيتوفر هنا قريباً." |

### 6. Edge Cases

| # | Scenario | Trigger | Expected Behavior | Response |
|---|---|---|---|---|
| E1 | QR generation fails | Image processing error | Retry 3 times (exponential backoff); if still fails, set org flag `qr_generation_failed = true`; alert admin | Admin sees "QR generation failed" in org record |
| E2 | Org deactivated after QR | Admin deactivates org | QR URL resolves to branded "inactive" landing page | "This location is no longer active" |
| E3 | QR scanned before marketplace live | Premature scan | URL resolves to "Coming soon" holding page with org name | Branded holding page |
| E4 | Regenerate QR | Admin requests regeneration | New QR generated; old QR URL still works (same encoded URL) | "New QR generated" confirmation |
| E5 | S3 upload fails | Network/permissions error | Retry 3 times; alert admin on failure | Admin notification |
| E6 | Pre-signed URL expired | User clicks download link after 7 days | GET endpoint generates fresh pre-signed URL on each request | Fresh URL returned |

### 7. Wave Dependencies

**Depends On:**
| Source | Consumes | Verification |
|---|---|---|
| Wave 1.1.1 | Organization record with `id`, GPS. Updates `qr_code_png_url` and `qr_code_svg_url` via PUT | Verify org record exists and accepts URL updates |
| Wave 1.1.2 | Approval event (`org.approved`) triggers QR generation | Approve a test org; verify event emitted |
| IDP | S3 bucket `tabsense-walk-qr-assets` exists; walk-worker pod deployed | Verify S3 bucket accessible from walk namespace |

**Exposes To:**
| Downstream | Consumes | Contract |
|---|---|---|
| Wave 1.2.1 (QR Scan Entry) | QR encodes URL `https://walk.tabsense.com/m/{org_id}`. Frontend at this URL is the marketplace entry point. | URL format must not change. The `{org_id}` is a UUID. |

**Route Registry:**
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/organizations/:id/qr-code` | POST | Internal: trigger generation |
| `/api/v1/organizations/:id/qr-code` | GET | Org admin: download URLs |
| `/api/v1/organizations/:id/qr-code/regenerate` | POST | Regeneration |
| `https://walk.tabsense.com/m/{org_id}` | GET | Public: marketplace entry (consumed by 1.2.1) |

### 8. QO Testing Guidance

**Risk Areas:** QR scan reliability across device types (iOS native camera, Android Google Lens, Samsung camera); URL encoding correctness with UUID special characters; print quality at various sizes.

**Test Focus:**
- Generate QR; scan on iOS Safari, Android Chrome, Samsung camera (3+ devices)
- Verify URL resolves to correct `org_id`
- Print QR at A4 and business card size; verify scannability
- Test deactivated org QR shows inactive page
- Test regeneration: old and new QR both resolve

**Approach:** Integration (generation pipeline end-to-end); manual (physical scan testing on real devices).

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

