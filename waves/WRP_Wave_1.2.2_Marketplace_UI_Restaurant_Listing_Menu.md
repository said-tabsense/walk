# Wave 1.2.2: Marketplace UI (Restaurant Listing + Menu Browse)

| Field | Value |
|---|---|
| **Feature** | Marketplace UI (Restaurant Listing + Menu Browse) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 3-4 hours |

### 1. Intent Definition

**Problem:** Users who scan the QR code need a visual, mobile-first interface to browse nearby restaurants and their menus. Without this UI, the API data has no consumer-facing representation.

**Outcome:** A responsive PWA page showing restaurant cards sorted by distance, with cuisine filters, search, and inline menu browsing. Tapping a restaurant shows its full menu with categories, items, modifiers, and prices. The UX targets "scan to browse in < 5 seconds."

**Metrics:**
- Page load < 2 seconds on 4G
- Menu browse with zero page reloads (SPA navigation)
- User can reach checkout from scan in < 60 seconds
- Layout works on 320px-428px mobile widths and desktop

**Constraints:**
- Single marketplace UI (not per-restaurant branded pages)
- Mobile-first; desktop is secondary
- Menu data from TabSense internal API via Wave 1.2.1 restaurant list

### 2. User Stories

**US-1: Browse Restaurant List**

```
GIVEN the marketplace page has loaded at walk.tabsense.com/m/{org_id}
WHEN the user views the page
THEN they see:
  - Header: org name + TabSense Walk branding
  - Restaurant cards sorted by distance: logo, name, cuisine, distance, walk time, rating, open/closed status
  - Group discount badge on restaurants offering discounts
  - Cuisine filter chips below header (e.g., "All", "Middle Eastern", "Asian", "Burgers")
  - Search bar with icon
```

**US-2: Browse Restaurant Menu**

```
GIVEN the user taps on a restaurant card
WHEN the menu view loads via GET /api/v1/marketplace/:org_id/restaurants/:restaurant_id/menu
THEN they see:
  - Restaurant header (logo, name, distance, walk time, rating)
  - Category tabs (horizontal scroll): e.g., "Appetizers", "Main Course", "Drinks"
  - Menu items: name, description, price (SAR), photo (if available), availability status
  - Unavailable items grayed with "Unavailable" label
  - "Back to restaurants" navigation (back arrow or breadcrumb)
  - Sticky "View Cart" bar at bottom (appears when items in cart)
```

**US-3: Filter and Search**

```
GIVEN the user is on the marketplace page
WHEN they tap a cuisine filter chip or type in the search bar
THEN the restaurant list filters/updates (client-side for cuisine filter; API search for dish search)
  AND search results show matching restaurants with highlighted dishes
  AND search is debounced at 300ms
```

### 3. API Contracts

**GET /api/v1/marketplace/:org_id/restaurants/:restaurant_id/menu**

| Property | Value |
|---|---|
| Method | GET |
| Path | `/api/v1/marketplace/:org_id/restaurants/:restaurant_id/menu` |
| Auth | None (public) |

**Response 200:**
```json
{
  "restaurant": {
    "id": "uuid",
    "name": "string",
    "name_ar": "string | null",
    "logo_url": "string",
    "estimated_walk_min": 4,
    "rating": 4.5
  },
  "categories": [
    {
      "id": "uuid",
      "name": "Appetizers",
      "name_ar": "المقبلات",
      "sort_order": 1,
      "items": [
        {
          "id": "uuid",
          "name": "Hummus",
          "name_ar": "حمص",
          "description": "Classic chickpea dip with tahini",
          "price": 15.00,
          "image_url": "string | null",
          "is_available": true,
          "modifiers": [
            {
              "id": "uuid",
              "name": "Add-ons",
              "name_ar": "إضافات",
              "options": [
                { "id": "uuid", "name": "Extra tahini", "name_ar": "طحينة إضافية", "price_adjustment": 3.00 }
              ],
              "required": false,
              "max_selections": 3
            }
          ]
        }
      ]
    }
  ]
}
```

**GET /api/v1/marketplace/:org_id/search?q=keyword**

| Property | Value |
|---|---|
| Method | GET |
| Auth | None (public) |

**Response 200:**
```json
{
  "restaurants": [ "...matching restaurant objects..." ],
  "items": [
    {
      "restaurant_id": "uuid",
      "restaurant_name": "string",
      "item": { "...MenuItem object..." }
    }
  ]
}
```

### 4. Wireframe Descriptions

**Screen: Marketplace Home (`/m/:org_id`)**

Layout: Vertical, mobile-first (max-width: 428px content area).

Components:
1. **Header (sticky, 60px):** TabSense Walk logo (left) + Org name (center) + Cart icon with badge (right)
2. **Search Bar (48px):** Search icon + "Search restaurants or dishes..." placeholder. Expands on focus.
3. **Filter Chips (horizontal scroll, 40px):** "All" (default selected), "Middle Eastern", "Asian", "Burgers", "Healthy", "Coffee" (derived from restaurant cuisine types)
4. **Restaurant Cards (repeating):** Each card 100px height, full width:
   - Left: Restaurant logo (64x64px, border-radius 8px)
   - Right column: Name (16px bold), cuisine (14px gray), distance+walk time row (14px, icon + "320m · ~4 min"), rating stars (14px)
   - Group discount badge (if applicable): green tag "10% off for 5+" (12px)
   - Closed state: card opacity 0.5, "Opens at 9:00 AM" overlay
5. **Empty State:** Walking icon illustration + "No restaurants available" message + "Notify me" button

**Design Tokens:**

| Token | Value |
|---|---|
| Page bg | #F9FAFB (gray-50) |
| Card bg | #FFFFFF |
| Card border-radius | 12px |
| Card shadow | 0 1px 3px rgba(0,0,0,0.1) |
| Card padding | 12px |
| Card gap | 12px between cards |
| Primary text | #111827 (gray-900) |
| Secondary text | #6B7280 (gray-500) |
| Price text | #059669 (emerald-600) |
| Rating stars | #F59E0B (amber-500) |
| Filter chip active | #2563EB bg, white text |
| Filter chip inactive | #F3F4F6 bg, #374151 text |
| Discount badge | #DCFCE7 bg, #166534 text |
| Cart icon badge | #EF4444 bg, white text, 20px circle |
| Sticky header bg | #FFFFFF, border-bottom 1px #E5E7EB |

**Screen: Menu View (`/m/:org_id/r/:restaurant_id`)**

Components:
1. **Back button + Restaurant header:** Back arrow, logo (80x80), name, cuisine, distance, rating
2. **Category tabs (sticky, horizontal scroll):** Tab per category, active tab underlined in blue
3. **Item list:** Per item: name (16px bold), description (14px gray, 2 lines max), price "SAR XX" (16px emerald), image thumbnail (60x60 if available), modifier count badge
4. **Item detail bottom sheet (on tap):** Full item details, modifier selection groups (radio for required, checkbox for optional), quantity control, "Add to Cart" button (primary blue, 48px)
5. **Sticky cart bar (bottom, 56px):** "View Cart ({count}) · SAR {total}" button (full width, primary blue)

### 5. Content Specification

| Key | English | Arabic |
|---|---|---|
| search_placeholder | "Search restaurants or dishes..." | "ابحث عن مطاعم أو أطباق..." |
| filter_all | "All" | "الكل" |
| distance_format | "{distance}m · ~{min} min" | "{distance}م · ~{min} دقيقة" |
| closed_overlay | "Opens at {time}" | "يفتح الساعة {time}" |
| no_results | "No results for '{query}'. Try a different search." | "لا توجد نتائج لـ '{query}'. جرّب بحثاً مختلفاً." |
| unavailable_label | "Unavailable" | "غير متوفر" |
| view_cart_btn | "View Cart ({count}) · SAR {total}" | "عرض السلة ({count}) · {total} ر.س" |
| back_to_restaurants | "Back to restaurants" | "العودة للمطاعم" |
| add_to_cart_btn | "Add to Cart" | "أضف للسلة" |
| price_format | "SAR {price}" | "{price} ر.س" |
| modifier_required | "Required" | "مطلوب" |
| modifier_optional | "Optional (up to {max})" | "اختياري (حتى {max})" |

### 6. Edge Cases

| # | Scenario | Trigger | Expected Behavior | Response |
|---|---|---|---|---|
| E1 | Menu item unavailable | Restaurant marks unavailable in POS | Grayed item, "Unavailable" label | Cannot add to cart |
| E2 | Restaurant closes mid-browse | Operating hours end | Banner: "This restaurant has closed" | Cart cleared for that restaurant |
| E3 | Search no results | User searches for nonexistent dish | Empty state | "No results for '[query]'" |
| E4 | Slow connection | > 3 seconds load | Skeleton loaders | Progressive loading |
| E5 | Image fails to load | Logo or menu item image 404 | Fallback placeholder image | Generic food/restaurant icon |

### 7. Wave Dependencies

**Depends On:** Wave 1.2.1 (Restaurant list API); TabSense internal API (menu data)
**Exposes To:** Wave 1.3.1 (Cart; user adds items from this UI)

**Route Registry:**
| Route | Type | Purpose |
|---|---|---|
| `/api/v1/marketplace/:org_id/restaurants/:restaurant_id/menu` | GET | Menu data |
| `/api/v1/marketplace/:org_id/search?q=` | GET | Search |
| Frontend: `/m/:org_id` | Page | Restaurant list |
| Frontend: `/m/:org_id/r/:restaurant_id` | Page | Menu view |

### 8. QO Testing Guidance

**Risk Areas:** Menu data sync freshness; responsive layout; RTL readiness; modifier display complexity.
**Test Focus:** E2E QR scan to menu browse; verify menu matches POS data; test on 3 screen sizes (320px, 375px, 428px); test search debounce; test closed restaurant handling.
**Approach:** E2E-heavy; manual for visual/responsive testing.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

