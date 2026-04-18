# Wave 1.2.4: User Auth + Organization Linking

| Field | Value |
|---|---|
| **Feature** | User Auth + Organization Linking |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Users who scan the QR code need to authenticate (for order tracking and group orders) and link to the organization for auto-filled delivery addresses. Guest checkout must also be supported for frictionless individual orders.

**Outcome:** Phone OTP authentication; auto-link user to organization whose QR they scanned; multi-org support for users with multiple workplaces; guest checkout for individual orders without account creation.

**Metrics:**
- OTP delivery < 10 seconds
- Auth flow completion < 30 seconds
- 100% correct org linking on QR scan
- Guest checkout available without auth

**Constraints:**
- Guest checkout allowed for individual orders only (name + phone)
- Auth required for group orders (participant tracking)
- User can be linked to multiple organizations
- Reuses existing TabSense SMS OTP service

### 2. User Stories

**US-1: Phone OTP Authentication**

```
GIVEN the user is on the marketplace and taps "Sign in"
WHEN they enter their Saudi phone number (+966) and tap "Send Code"
THEN an OTP is sent via SMS (existing TabSense OTP service),
  the user enters the 6-digit code,
  AND on successful verification:
  - If new user: account created with phone number, JWT issued
  - If existing user: JWT issued with existing user data
  - User auto-linked to the current org (from QR scan context)
```

```
GIVEN the user enters the wrong OTP 3 times
WHEN the 3rd attempt fails
THEN the account is locked for 15 minutes:
  { "error": "too_many_attempts", "message": "Too many attempts. Try again in 15 minutes.", "retry_after": 900 }
```

**US-2: Guest Checkout**

```
GIVEN the user is browsing the marketplace without signing in
WHEN they proceed to checkout for an individual order
THEN they can complete the order by providing only name and phone number (for delivery contact),
  no account is created,
  the org delivery address is auto-filled from the org record,
  AND the order is linked to the guest by phone number.
```

**US-3: Multi-Org Linking**

```
GIVEN the user is authenticated and linked to 2+ organizations (e.g., scanned QR at two different offices)
WHEN they open the marketplace
THEN they see their current org context in the header,
  tapping it shows a switcher listing all linked orgs,
  selecting one reloads the marketplace for that org's location.
```

### 3. API Contracts

**POST /api/v1/auth/otp/request**
```
Auth: None
Request: { "phone": "+966XXXXXXXXX" }
Response 200: { "message": "OTP sent", "expires_in_seconds": 300 }
Errors: 429 (too many attempts)
```

**POST /api/v1/auth/otp/verify**
```
Auth: None
Request: { "phone": "+966XXXXXXXXX", "otp": "123456", "org_id": "uuid (optional, from QR context)" }
Response 200: {
  "token": "JWT string",
  "user": {
    "id": "uuid",
    "phone": "+966XXXXXXXXX",
    "name": "string | null",
    "linked_organizations": [
      { "id": "uuid", "name": "ACME Corp", "is_current": true }
    ]
  }
}
Errors: 401 (wrong OTP), 429 (locked 15 min)
```

**POST /api/v1/users/:id/link-organization**
```
Auth: User JWT
Request: { "org_id": "uuid" }
Response 200: { "user_id": "uuid", "linked_organizations": [...] }
Errors: 404 (org not found), 409 (already linked)
```

**DELETE /api/v1/users/:id/unlink-organization/:org_id**
```
Auth: User JWT
Response 200: { "message": "Unlinked", "remaining_organizations": [...] }
```

**PUT /api/v1/users/:id/current-organization**
```
Auth: User JWT
Request: { "org_id": "uuid" }
Response 200: { "current_org_id": "uuid" }
```

### 4. Wireframe Descriptions

**Auth Flow (Bottom Sheet):**
1. "Sign in" tapped → Bottom sheet slides up (80% screen height)
2. Phone input with +966 prefix pre-filled
3. "Send Code" button → OTP digit inputs appear (6 boxes)
4. Countdown timer + "Resend" link
5. On success: sheet dismisses, user avatar/icon appears in header

**Org Switcher:**
- Tapping org name in header opens a dropdown/bottom sheet
- List of linked orgs with radio buttons
- "Current" badge on active org
- Selecting different org reloads marketplace data

### 5. Content Specification

| Key | English | Arabic |
|---|---|---|
| sign_in_btn | "Sign in" | "تسجيل الدخول" |
| phone_label | "Phone Number" | "رقم الهاتف" |
| send_code_btn | "Send Code" | "إرسال الرمز" |
| otp_instruction | "Enter the 6-digit code sent to {phone}" | "أدخل الرمز المكون من 6 أرقام المرسل إلى {phone}" |
| wrong_otp | "Incorrect code. Please try again." | "رمز غير صحيح. يرجى المحاولة مرة أخرى." |
| too_many_attempts | "Too many attempts. Try again in 15 minutes." | "محاولات كثيرة. حاول مرة أخرى بعد 15 دقيقة." |
| guest_checkout_label | "Continue as guest" | "المتابعة كضيف" |
| org_switcher_title | "Switch Organization" | "تبديل المنشأة" |
| current_badge | "Current" | "الحالية" |

### 6. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | OTP wrong 3x | Lock 15 minutes |
| E2 | Scan different org QR while signed in | Auto-link to new org; switch context |
| E3 | Guest tries group order | Prompt auth: "Sign in to join group order" |
| E4 | Existing TabSense user scans QR | Merge: add org link to existing account |
| E5 | User unlinks only org | Allow; user becomes org-less (can re-link by scanning QR) |

### 7. Wave Dependencies

**Depends On:** Wave 1.1.1 (Organization data for linking); TabSense OTP service
**Exposes To:** Wave 1.3.2 (Checkout uses org address); Wave 2.1.1 (Group orders require auth)
**Database:** `user_organizations` junction table (IDP schema)

**Route Registry:**
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/auth/otp/request` | POST | Send OTP |
| `/api/v1/auth/otp/verify` | POST | Verify OTP + issue JWT |
| `/api/v1/users/:id/link-organization` | POST | Link user to org |
| `/api/v1/users/:id/unlink-organization/:org_id` | DELETE | Unlink |
| `/api/v1/users/:id/current-organization` | PUT | Switch active org |

### 8. QO Testing Guidance

**Risk Areas:** OTP delivery reliability; multi-org context switching; guest-to-authenticated upgrade mid-session.
**Test Focus:** Full auth E2E; multi-org link and switch; guest checkout bypass; existing TabSense user merge.
**Approach:** E2E-heavy; unit for JWT claims and org context.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

