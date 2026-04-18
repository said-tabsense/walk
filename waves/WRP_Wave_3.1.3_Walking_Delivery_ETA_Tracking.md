# Wave 3.1.3: Walking Delivery ETA Tracking

| Field | Value |
|---|---|
| **Feature** | Walking Delivery Status + ETA |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Users need real-time order status updates with estimated walking delivery time.

**Outcome:** Order status page with progress tracker (Confirmed → Preparing → Walking → Delivered). ETA calculated from walking distance. Real-time updates when restaurant changes status.

**Metrics:**
- Status update latency < 5 seconds
- ETA accuracy within 2 minutes (based on walking distance / 80m per min)

### 2. API Contracts

**PUT /api/v1/marketplace/orders/:id/status**
```
Auth: Merchant JWT or POS system
Request: { "status": "preparing | ready | walking | delivered" }
Response 200: { "order_id", "status", "estimated_delivery_min", "updated_at" }
Broadcast: Push notification + WebSocket update
```

### 3. Wireframe Description

**Order Status Page (`/m/:org_id/order/:order_id`):**
- Progress bar with 4 steps: Confirmed → Preparing → Walking → Delivered
- Current step highlighted (blue fill), completed steps (green check), future steps (gray)
- ETA displayed when status = "walking": "Arriving in ~{min} minutes"
- Order details below: items, total, restaurant name

### 4. Content Specification

| Key | English | Arabic |
|---|---|---|
| confirmed | "Order Confirmed" | "تم تأكيد الطلب" |
| preparing | "Preparing" | "قيد التحضير" |
| walking | "On the Way" | "في الطريق" |
| delivered | "Delivered" | "تم التوصيل" |
| eta_message | "Arriving in ~{min} minutes" | "يصل خلال ~{min} دقيقة" |

### 5. Wave Dependencies

**Depends On:** Wave 1.3.4 (POS sync; order exists); Wave 1.2.1 (Distance for ETA)
**Exposes To:** Wave 3.2.1 (Status state machine); Wave 3.2.2 (Push notifications)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

