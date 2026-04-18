# Wave 2.1.1: Group Order Session API

| Field | Value |
|---|---|
| **Feature** | Group Order Session API |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Group orders require a session that multiple users can join, each adding their own items. There is no multi-user ordering concept in the current system.

**Outcome:** CRUD API for group order sessions: create, join, add/remove items, close, and get state. Real-time updates via WebSocket so all participants see changes live. Session has a 2-hour expiration.

**Metrics:**
- Session creation < 500ms
- Real-time WebSocket update latency < 1 second
- Supports up to 50 concurrent participants per session
- Session auto-expires after 2 hours if not submitted

### 2. User Stories

**US-1: Create Group Order Session**
```
GIVEN user is authenticated and on a restaurant's menu
WHEN they tap "Start Group Order"
THEN session created with user as organizer, share link generated, status "open".
```

**US-2: Join Group Order**
```
GIVEN user receives group order link and authenticates
WHEN they join the session
THEN they see current state: participants, items, running total. WebSocket connection established.
```

**US-3: Add Items as Participant**
```
GIVEN user has joined a session
WHEN they add menu items
THEN items appear in shared cart tagged with their name. All participants see update in real-time.
```

**US-4: Close and Submit**
```
GIVEN organizer views group order
WHEN they tap "Close & Submit"
THEN no more items can be added; session moves to checkout; discount auto-applied if applicable.
```

### 3. API Contracts

**POST /api/v1/marketplace/group-orders** (Create)
```
Auth: User JWT
Request: { "organization_id": "uuid", "restaurant_id": "uuid" }
Response 201: { "session_id", "share_link": "https://walk.tabsense.com/g/{session_id}", "organizer", "status": "open", "expires_at" }
```

**POST /api/v1/marketplace/group-orders/:session_id/join**
```
Auth: User JWT
Response 200: { "session_id", "participant", "participants_count", "status" }
Errors: 404, 410 (expired), 409 (already joined/full), 403 (closed)
```

**POST /api/v1/marketplace/group-orders/:session_id/items** (Add item)
```
Auth: User JWT (participant)
Request: { "menu_item_id", "quantity", "modifiers", "special_instructions" }
Response 201: { "item_id", "added_by", "item_total" }
Broadcast: WebSocket event to all participants
```

**DELETE /api/v1/marketplace/group-orders/:session_id/items/:item_id**
```
Auth: User JWT (item owner or organizer)
```

**PUT /api/v1/marketplace/group-orders/:session_id/close** (Organizer only)
```
Auth: User JWT (organizer)
Response 200: { "session_id", "status": "closed", "summary": { participants_count, items_count, subtotal, discount, total } }
```

**GET /api/v1/marketplace/group-orders/:session_id**
```
Auth: User JWT (any participant)
Response 200: Full session state
```

**WebSocket:** `wss://walk-api.tabsense.com/ws/group-orders/:session_id`
Events: `participant_joined`, `participant_left`, `item_added`, `item_removed`, `session_closed`, `session_expired`

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| start_group_order | "Start Group Order" | "بدء طلب جماعي" |
| join_group_order | "Join Group Order" | "الانضمام لطلب جماعي" |
| session_expired | "This group order has expired." | "انتهت صلاحية هذا الطلب الجماعي." |
| session_full | "This group order is full (max 50)." | "هذا الطلب الجماعي ممتلئ (الحد الأقصى 50)." |
| close_submit | "Close & Checkout" | "إغلاق والدفع" |
| running_total | "Running Total" | "المجموع الحالي" |
| participants_label | "{count} participants" | "{count} مشاركين" |
| item_unavailable_ws | "{item} is no longer available. Please choose something else." | "{item} لم يعد متوفراً. يرجى اختيار شيء آخر." |

### 5. Database

Uses `group_order_sessions` and `group_order_items` tables from IDP schema.

### 6. Wave Dependencies

**Depends On:** Wave 1.2.4 (Auth required); Wave 1.2.2 (Menu data); Wave 1.3.1 (Cart logic adapted)
**Exposes To:** Wave 2.1.2 (Group UI); Wave 2.1.4 (Payment modes); Wave 2.2.2 (Discount auto-apply)

**Route Registry:**
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/marketplace/group-orders` | POST | Create session |
| `/api/v1/marketplace/group-orders/:id/join` | POST | Join |
| `/api/v1/marketplace/group-orders/:id/items` | POST | Add item |
| `/api/v1/marketplace/group-orders/:id/items/:item_id` | DELETE | Remove item |
| `/api/v1/marketplace/group-orders/:id/close` | PUT | Close |
| `/api/v1/marketplace/group-orders/:id` | GET | Get state |
| `wss://walk-api.tabsense.com/ws/group-orders/:id` | WS | Real-time |
| Frontend: `/g/:session_id` | Page | Group order view |

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

