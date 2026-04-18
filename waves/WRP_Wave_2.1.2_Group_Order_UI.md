# Wave 2.1.2: Group Order UI

| Field | Value |
|---|---|
| **Feature** | Group Order UI (Organizer + Participant Views) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Group orders need a real-time collaborative UI where organizer and participants see the shared cart, item attribution, and running totals.

**Outcome:** Two role-detected views: Organizer view (with close/submit controls) and Participant view (browse + add own items). Both update in real-time via WebSocket.

**Metrics:**
- Real-time updates visible within 1 second
- Clear item-to-participant attribution
- Organizer controls not visible to participants

### 2. User Stories

**US-1: Organizer View**
```
GIVEN organizer is on the group order page
WHEN participants add items
THEN organizer sees live-updating shared cart grouped by participant:
  participant name, their items, individual subtotals, group subtotal, participant count, "Close & Checkout" button.
```

**US-2: Participant View**
```
GIVEN participant joined the group order
WHEN they view the session
THEN they see restaurant menu + sidebar/section with shared cart.
  They can add/remove their own items only. Group total updates live.
```

**US-3: Closed State**
```
GIVEN organizer closed the session
WHEN participants view
THEN "Order closed by [Organizer]. Waiting for checkout..." with final summary. No edits.
```

### 3. Wireframe Descriptions

**Organizer View (`/g/:session_id`):**
- Split layout (mobile: tabs; desktop: side-by-side)
- Left/Tab 1: Restaurant menu (same as Wave 1.2.2)
- Right/Tab 2: Shared cart grouped by participant. Each participant section: avatar + name, their items, their subtotal. Group subtotal at bottom. "Close & Checkout" primary button.
- Participant count badge in header
- Share button (triggers Wave 2.1.3)

**Participant View (`/g/:session_id`):**
- Same layout but: no "Close & Checkout" button; can only remove own items; "Waiting for organizer to submit" footer.

**Closed State:**
- Read-only summary. Progress indicator: "Organizer is completing checkout..."

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| organizer_label | "You (Organizer)" | "أنت (المنظم)" |
| participant_subtotal | "{name}'s items: SAR {total}" | "عناصر {name}: {total} ر.س" |
| group_subtotal | "Group Subtotal" | "المجموع الجماعي" |
| waiting_checkout | "Waiting for organizer to submit..." | "في انتظار المنظم لإرسال الطلب..." |
| closed_by | "Order closed by {name}" | "تم إغلاق الطلب بواسطة {name}" |
| share_btn | "Share" | "مشاركة" |
| no_items_yet | "No items added yet. Start browsing the menu!" | "لم تتم إضافة عناصر بعد. ابدأ بتصفح القائمة!" |

### 5. Wave Dependencies

**Depends On:** Wave 2.1.1 (Group order API + WebSocket)
**Exposes To:** Wave 2.1.3 (Share link UI); Wave 2.1.4 (Payment mode selection)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

