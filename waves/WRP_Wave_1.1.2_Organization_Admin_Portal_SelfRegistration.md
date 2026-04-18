# Wave 1.1.2: Organization Admin Portal (Self-Registration)

| Field | Value |
|---|---|
| **Feature** | Organization Admin Portal (Self-Registration) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Organizations need a way to self-register on TabSense Walk without depending on the TabSense sales team. Currently there is no public-facing registration flow, which limits onboarding velocity to whatever the sales team can handle manually.

**Outcome:** A public web form at `walk.tabsense.com/register` where an organization representative can register their company, input location via map pin, upload access media, and verify their identity via phone OTP. Submissions enter a `pending_review` queue. TabSense admins can approve or reject from the admin portal, triggering QR code generation on approval.

**Metrics:**
- Form completion rate > 70% (from start to OTP submission)
- Admin review turnaround < 24 hours
- Zero submissions lost (100% persistence on submit)
- OTP delivery < 10 seconds
- Form load time < 2 seconds on 4G

**Constraints:**
- Public-facing; no auth required to start the form
- Phone OTP verification required to submit
- Submissions create organization records with `status: pending_review`
- GPS must be selected via map pin (Google Maps embed or Mapbox), not manual lat/lng entry
- Riyadh-only geo-fence for MVP (reject locations outside Riyadh metro area)

### 2. User Stories

**US-1: Self-Register Organization**

**As an** office manager, **I want to** register my organization on TabSense Walk **so that** our employees can order food from nearby restaurants.

```
GIVEN the user is on walk.tabsense.com/register
WHEN they fill in: organization name, select building location on map (GPS pin),
  write delivery directions, optionally upload access photos (max 5, max 10MB each),
  enter contact person name, phone (+966 format), and email (optional)
  AND verify their phone via OTP
THEN the organization is created with status "pending_review",
  onboarding_channel = "self_registration",
  reference_number generated (format: "WALK-2026-XXXX"),
  AND the user receives an SMS: "Your TabSense Walk registration (Ref: WALK-2026-XXXX) is under review. You'll hear from us within 24 hours."
  AND the API returns 201 with reference_number and confirmation message.
```

```
GIVEN the user places the map pin outside the Riyadh metro geo-fence
WHEN they attempt to submit
THEN the API returns 422 with error:
  { "error": "geo_restriction", "message": "TabSense Walk is currently available in Riyadh only." }
  AND the map shows the supported boundary overlay.
```

```
GIVEN the user's OTP verification token has expired (> 5 minutes)
WHEN they attempt to submit the form
THEN the API returns 401 with error:
  { "error": "otp_expired", "message": "Your verification code has expired. Please request a new one." }
```

```
GIVEN a submission already exists with the same phone number and status "pending_review"
WHEN the user attempts to submit again
THEN the API returns 409 with error:
  { "error": "duplicate_submission", "message": "A registration from this phone is already under review (Ref: WALK-2026-XXXX)." }
```

**US-2: Admin Reviews Submission**

**As a** TabSense admin, **I want to** review pending organization registrations **so that** I can approve or reject them before QR codes are generated.

```
GIVEN there are organizations with status "pending_review"
WHEN the admin opens the review queue at the admin portal
THEN they see a list of pending submissions showing: org name, GPS on mini-map,
  contact person, phone, submission date, reference number.
  Each entry has "Approve" and "Reject" buttons.
```

```
GIVEN the admin clicks "Approve" on a submission
WHEN the approval is processed
THEN the organization status changes to "active",
  QR code generation is triggered (Wave 1.1.3),
  the contact person receives SMS: "Congratulations! Your organization [Name] is now active on TabSense Walk. Your QR code will be sent shortly."
```

```
GIVEN the admin clicks "Reject" on a submission
WHEN they provide a rejection reason (required field, min 10 chars)
THEN the organization status changes to "rejected",
  the contact person receives SMS: "Your TabSense Walk registration was not approved. Reason: [reason]. Contact support@tabsense.com for questions."
```

### 3. API Contracts

**POST /api/v1/organizations/register**

| Property | Value |
|---|---|
| Method | POST |
| Path | `/api/v1/organizations/register` |
| Auth | None (public) + OTP verification token in body |

**Request Body:**
```json
{
  "name": "string (required, max 200)",
  "name_ar": "string (optional, max 200)",
  "gps_lat": "number (required, within Riyadh geo-fence)",
  "gps_lng": "number (required, within Riyadh geo-fence)",
  "delivery_directions": "string (required, max 1000)",
  "delivery_directions_ar": "string (optional, max 1000)",
  "contact_person_name": "string (required, max 100)",
  "contact_person_phone": "string (required, +966XXXXXXXXX)",
  "contact_person_email": "string (optional, valid email)",
  "access_media_urls": "string[] (optional, max 5)",
  "otp_verification_token": "string (required)"
}
```

**Response 201:**
```json
{
  "id": "uuid",
  "status": "pending_review",
  "reference_number": "WALK-2026-0001",
  "message": "Your registration is under review. You'll receive an SMS within 24 hours."
}
```

**Error Responses:**

| Code | Condition | Body |
|---|---|---|
| 400 | Validation failure | `{ "error": "validation_error", "fields": { ... } }` |
| 401 | OTP expired/invalid | `{ "error": "otp_expired", "message": "Your verification code has expired." }` |
| 409 | Duplicate phone pending | `{ "error": "duplicate_submission", "message": "A registration from this phone is already under review (Ref: ...)." }` |
| 422 | Outside Riyadh geo-fence | `{ "error": "geo_restriction", "message": "TabSense Walk is currently available in Riyadh only." }` |
| 422 | Duplicate GPS (< 50m) | Same as Wave 1.1.1 duplicate_location error |

**PUT /api/v1/admin/organizations/:id/review**

| Property | Value |
|---|---|
| Method | PUT |
| Path | `/api/v1/admin/organizations/:id/review` |
| Auth | TabSense Admin JWT |

**Request Body:**
```json
{
  "action": "enum: approve | reject",
  "rejection_reason": "string (required if reject, min 10 chars, max 500)"
}
```

**Response 200:**
```json
{
  "id": "uuid",
  "status": "active | rejected",
  "reviewed_by": "admin_name",
  "reviewed_at": "ISO8601"
}
```

**GET /api/v1/admin/organizations?status=pending_review**

Uses the existing Wave 1.1.1 list endpoint with status filter. No new endpoint needed.

### 4. Wireframe Descriptions

**Screen: Self-Registration Form (`walk.tabsense.com/register`)**

Layout: Single-page form, mobile-optimized, vertical scroll. TabSense Walk branding at top.

Components (top to bottom):
1. **Header:** TabSense Walk logo + "Register Your Organization" / "سجّل منشأتك"
2. **Organization Name Input:** Text field, required, placeholder "e.g., ACME Corp Office"
3. **Location Map:** Embedded map (300px height on mobile, 400px desktop). User taps to place pin. Riyadh geo-fence boundary shown as subtle overlay. Below map: "Selected coordinates: 24.xxxx, 46.xxxx" (auto-filled, read-only)
4. **Delivery Directions Textarea:** 3 rows, required, placeholder "How should delivery staff find your office? e.g., Building C, ground floor reception"
5. **Access Media Upload:** Drag-drop zone or "Upload Photos" button. Grid preview of uploaded images. Max 5, max 10MB each. Accepted: jpg, png, webp
6. **Contact Section:** Name input + Phone input (+966 prefix) + Email input (optional)
7. **OTP Section:** "Verify Phone" button. On click: OTP sent, input field appears for 6-digit code, 5-minute countdown timer. "Resend OTP" link after 60 seconds.
8. **Submit Button:** "Submit Registration" / "تقديم التسجيل". Disabled until OTP verified. Full width, primary blue, 48px height.
9. **Footer:** "Already registered? Check your status" link + TabSense support email

**Design Tokens:**

| Token | Value |
|---|---|
| Page max-width | 600px (centered) |
| Page padding | 20px horizontal |
| Input height | 44px |
| Input border-radius | 8px |
| Input border | 1px solid #D1D5DB |
| Input focus border | 2px solid #2563EB |
| Label font | 14px, weight 500, #374151 |
| Section gap | 24px |
| Map border-radius | 12px |
| OTP input | 6 separate digit boxes, 48x48px each, centered |
| Submit button bg | #2563EB |
| Submit button text | white, 16px, weight 600 |
| Success state bg | #F0FDF4 (green-50) |
| Error state bg | #FEF2F2 (red-50) |

**UI States:**
- Default: Empty form, submit disabled
- OTP Sent: Phone field locked, OTP input visible, countdown timer
- OTP Verified: Green checkmark on phone field, submit enabled
- Submitting: Spinner on submit button, all fields disabled
- Success: Form replaced with success card: "Registration submitted! Ref: WALK-2026-XXXX"
- Error: Red banner at top with specific error message, scroll to first error field

**Screen: Admin Review Queue**

Location: TabSense Admin Portal → Organizations → Pending Review tab

Layout: Table view with expandable rows.
- Columns: Reference # | Org Name | Location (mini-map thumbnail) | Contact | Submitted | Actions
- Actions: "Approve" (green button) + "Reject" (red outline button)
- Reject modal: textarea for reason (required), "Confirm Rejection" button
- Approve: confirmation dialog "Approve [Org Name]? This will generate a QR code." → "Yes, Approve"

### 5. Content Specification

**Registration Form UI Strings:**

| Key | English | Arabic |
|---|---|---|
| page_title | "Register Your Organization" | "سجّل منشأتك" |
| page_subtitle | "Get TabSense Walk for your office" | "احصل على TabSense Walk لمكتبك" |
| name_label | "Organization Name" | "اسم المنشأة" |
| name_placeholder | "e.g., ACME Corp Office" | "مثال: مكتب شركة أكمي" |
| map_label | "Pin Your Location" | "حدد موقعك" |
| map_instruction | "Tap the map to place your pin" | "انقر على الخريطة لتحديد موقعك" |
| directions_label | "Delivery Directions" | "تعليمات التوصيل" |
| directions_placeholder | "How should delivery staff find your office?" | "كيف يصل موظف التوصيل لمكتبك؟" |
| media_label | "Access Photos (optional)" | "صور الوصول (اختياري)" |
| media_instruction | "Upload up to 5 photos showing how to reach your office" | "ارفع حتى 5 صور توضح كيفية الوصول لمكتبك" |
| contact_name_label | "Contact Person Name" | "اسم جهة الاتصال" |
| contact_phone_label | "Phone Number" | "رقم الهاتف" |
| contact_email_label | "Email (optional)" | "البريد الإلكتروني (اختياري)" |
| verify_phone_btn | "Verify Phone" | "تحقق من الهاتف" |
| otp_instruction | "Enter the 6-digit code sent to your phone" | "أدخل الرمز المكون من 6 أرقام المرسل لهاتفك" |
| otp_resend | "Resend code" | "إعادة إرسال الرمز" |
| otp_countdown | "Resend in {seconds}s" | "إعادة الإرسال خلال {seconds}ث" |
| submit_btn | "Submit Registration" | "تقديم التسجيل" |
| success_title | "Registration Submitted!" | "تم تقديم التسجيل!" |
| success_message | "Your reference number is {ref}. You'll hear from us within 24 hours." | "رقمك المرجعي هو {ref}. سنتواصل معك خلال 24 ساعة." |
| geo_error | "TabSense Walk is currently available in Riyadh only" | "TabSense Walk متوفر حالياً في الرياض فقط" |
| otp_expired | "Your verification code has expired. Please request a new one." | "انتهت صلاحية رمز التحقق. يرجى طلب رمز جديد." |
| duplicate_phone | "A registration from this phone is already under review" | "يوجد تسجيل من هذا الرقم قيد المراجعة" |
| file_too_large | "Files must be under 10MB each" | "يجب أن يكون حجم كل ملف أقل من 10 ميجا" |
| max_files | "Maximum 5 files allowed" | "الحد الأقصى 5 ملفات" |

**SMS Templates:**

| Event | English |
|---|---|
| OTP | "Your TabSense Walk verification code is: {code}. Valid for 5 minutes." |
| Submission Confirmation | "Your TabSense Walk registration (Ref: {ref}) is under review. You'll hear from us within 24 hours." |
| Approval | "Congratulations! Your organization {name} is now active on TabSense Walk. Your QR code will be sent shortly." |
| Rejection | "Your TabSense Walk registration was not approved. Reason: {reason}. Contact support@tabsense.com for questions." |

### 6. Edge Cases

| # | Scenario | Trigger | Expected Behavior | Response |
|---|---|---|---|---|
| E1 | OTP expires | > 5 min after OTP sent | Token invalid | 401: "Verification code expired" |
| E2 | Duplicate phone pending | Same phone, existing pending_review | Block submission | 409: "Already under review" |
| E3 | GPS outside Riyadh | Pin placed outside geo-fence | Block submission | 422: "Available in Riyadh only" |
| E4 | Duplicate GPS (< 50m) | Same location as existing org | Block submission | 422: "Organization exists near this location" |
| E5 | Admin rejects without reason | Empty reason field | Frontend blocks | "Please provide a reason for rejection" |
| E6 | File too large | Single file > 10MB | Frontend blocks upload | "Files must be under 10MB each" |
| E7 | Too many files | > 5 files selected | Frontend blocks | "Maximum 5 files allowed" |
| E8 | OTP wrong 3 times | 3 incorrect attempts | Lock for 15 min | "Too many attempts. Try again in 15 minutes." |
| E9 | Network failure on submit | Connection drops | Retry button appears | "Submission failed. Tap to retry." |
| E10 | Form refresh mid-fill | User refreshes browser | Form state lost (MVP) | Empty form; no auto-save for MVP |

### 7. Wave Dependencies

**Depends On:**
| Source | Consumes | Verification |
|---|---|---|
| Wave 1.1.1 | `organizations` table schema; POST creates records with `status: pending_review` | Existing table from Wave 1.1.1 accepts new records |
| TabSense OTP Service | `POST http://tabsense-api.default.svc.cluster.local:8080/internal/otp/send` and `/verify` | Call OTP send endpoint with test phone number |

**Exposes To:**
| Downstream | Consumes | Contract |
|---|---|---|
| Wave 1.1.3 (QR Gen) | Approval event triggers QR generation | On approval, emit event `org.approved` with org ID |
| Wave 1.1.4 (Onboarding) | Referral path creates similar pending entries | Same `organizations` table, same `pending_review` status |

**Route Registry:**
| Route | Method | File | Purpose |
|---|---|---|---|
| `/api/v1/organizations/register` | POST | `routes/api.php` or `routes/organizations.ts` | Public registration |
| `/api/v1/admin/organizations/:id/review` | PUT | Same file | Admin approve/reject |
| Frontend: `/register` | GET | `pages/register.tsx` | Self-registration form |

### 8. QO Testing Guidance

**Risk Areas:** OTP race conditions (two users verifying simultaneously); duplicate detection under concurrent submissions; Riyadh geo-fence boundary accuracy.

**Test Focus:**
- E2E: Complete registration flow from form fill to admin approval to status change to SMS delivery
- OTP timing: verify expiration at exactly 5 minutes; verify 3-attempt lockout
- Geo-fence: test coordinates just inside and just outside Riyadh boundary
- Duplicate detection: submit two registrations with phones 1 second apart

**Approach:** E2E for happy path; unit tests for OTP validation and geo-boundary checks; integration for SMS delivery verification.

**Riyadh Geo-fence Definition (for implementation):**
Bounding box (approximate): Lat 24.4 to 25.1, Lng 46.2 to 47.0. For MVP, a simple bounding box is acceptable; upgrade to polygon for precision later.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

