# Wave 1.3.1: Cart + Menu Selection (Multi-Restaurant Aware)

| Field | Value |
|---|---|
| **Feature** | Cart + Menu Selection |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Users need to add menu items (with modifiers) to a cart and manage quantities before checkout. The cart must be restaurant-specific; mixing items from different restaurants is not allowed.

**Outcome:** Client-side cart (React state/Zustand) supporting add/remove/update items with modifiers, accurate price calculation, session persistence, and restaurant-switch warning.

**Metrics:**
- Add-to-cart interaction < 200ms
- Cart state persists across page navigation within session
- Price calculation accurate to 2 decimal places including modifiers
- Zero cross-restaurant contamination

**Constraints:**
- Cart is client-side only (no cart API)
- One active cart per restaurant
- Session storage for persistence (not localStorage per artifact constraints)
- Cart data structure must be compatible with Wave 1.3.2 checkout submission

### 2. User Stories

**US-1: Add Item to Cart**
```
GIVEN the user is viewing a restaurant's menu item detail (bottom sheet)
WHEN they select required modifiers, optionally select optional modifiers, set quantity, and tap "Add to Cart"
THEN the item with modifiers and calculated price is added to cart state,
  cart badge updates, toast: "Added to cart."
```

**US-2: Switch Restaurant Warning**
```
GIVEN cart has items from Restaurant A
WHEN user adds item from Restaurant B
THEN dialog: "You have items from [Restaurant A]. Starting a new order will clear your cart. Continue?"
  "Clear Cart" → clears and adds new item; "Cancel" → stays on current page.
```

**US-3: View and Edit Cart**
```
GIVEN user taps cart icon or sticky "View Cart" bar
WHEN cart page loads
THEN shows: item names with modifiers, quantities (adjustable +/-), individual prices, subtotal.
  User can remove items (swipe or X button). "Proceed to Checkout" button at bottom.
```

### 3. API Contracts

No API needed. Cart is fully client-side.

**Price Calculation:**
```
item_total = item.price + SUM(selected_modifiers.price_adjustment)
line_total = item_total * quantity
cart_subtotal = SUM(line_total) for all items
```

**Cart Data Structure (client state):**
```json
{
  "restaurant_id": "uuid",
  "restaurant_name": "string",
  "org_id": "uuid",
  "items": [
    {
      "cart_item_id": "uuid (client-generated)",
      "menu_item_id": "uuid",
      "name": "string",
      "base_price": 15.00,
      "modifiers": [{ "modifier_id": "uuid", "option_id": "uuid", "name": "string", "price_adjustment": 3.00 }],
      "quantity": 2,
      "special_instructions": "string | null",
      "item_total": 18.00,
      "line_total": 36.00
    }
  ],
  "subtotal": 36.00,
  "item_count": 2
}
```

### 4. Wireframe Descriptions

**Cart Page (`/m/:org_id/r/:restaurant_id/cart`):**
- Header: "Your Cart" + restaurant name
- Item list: each row has item name, modifiers (small gray text), quantity controls (-/+), line total, X remove button
- Subtotal row: bold, right-aligned
- "Proceed to Checkout" button: full width, primary blue, 48px
- Empty state: "Your cart is empty. Browse nearby restaurants to get started."

### 5. Content Specification

| Key | English | Arabic |
|---|---|---|
| cart_title | "Your Cart" | "سلتك" |
| empty_cart | "Your cart is empty" | "سلتك فارغة" |
| empty_cart_cta | "Browse nearby restaurants to get started" | "تصفح المطاعم القريبة للبدء" |
| switch_restaurant_title | "Start a new order?" | "بدء طلب جديد؟" |
| switch_restaurant_msg | "You have items from {restaurant}. Starting a new order will clear your cart." | "لديك عناصر من {restaurant}. بدء طلب جديد سيمسح سلتك." |
| clear_cart_btn | "Clear Cart" | "مسح السلة" |
| cancel_btn | "Cancel" | "إلغاء" |
| proceed_checkout | "Proceed to Checkout" | "المتابعة للدفع" |
| added_toast | "Added to cart" | "تمت الإضافة للسلة" |
| subtotal_label | "Subtotal" | "المجموع الفرعي" |
| modifier_required_error | "Please select {modifier_name}" | "يرجى اختيار {modifier_name}" |
| item_unavailable_warning | "{count} item(s) in your cart are no longer available. Please remove to continue." | "{count} عنصر في سلتك لم يعد متوفراً. يرجى إزالته للمتابعة." |

### 6. Edge Cases

| # | Scenario | Expected Behavior |
|---|---|---|
| E1 | Item becomes unavailable after adding | Flag in cart; block checkout until removed |
| E2 | Price changed after adding | Update price on next cart view; show diff |
| E3 | Cart survives page refresh | Restore from session storage |
| E4 | Required modifier not selected | Block add-to-cart; show error |
| E5 | Quantity set to 0 | Remove item from cart |
| E6 | Cart at 50 items | Allow but show warning |

### 7. Wave Dependencies

**Depends On:** Wave 1.2.2 (Menu UI and item data)
**Exposes To:** Wave 1.3.2 (Checkout consumes cart data)

**Route Registry:**
| Route | Type | Purpose |
|---|---|---|
| Frontend: `/m/:org_id/r/:restaurant_id/cart` | Page | Cart view |

### 8. QO Testing Guidance

**Risk Areas:** Price calculation with complex modifiers; restaurant switch logic; session persistence.
**Test Focus:** Add items with 0, 1, 3 modifiers; verify subtotal; test restaurant switch; test persistence.
**Approach:** Unit-heavy for price calculation; E2E for cart interaction.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

