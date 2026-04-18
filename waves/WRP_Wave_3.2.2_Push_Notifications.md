# Wave 3.2.2: Push Notifications

| Field | Value |
|---|---|
| **Feature** | Push Notifications (Order Updates) |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Users need proactive notifications about order status changes without checking the app.

**Outcome:** PWA push notifications on key status transitions. Web Push API with service worker registration.

**Metrics:**
- Notification delivery < 10 seconds of status change
- Opt-in rate tracked
- Notification open rate tracked

### 2. User Stories

**US-1: Status Notifications**
```
GIVEN user placed an order with notifications enabled
WHEN status changes to preparing, walking, or delivered
THEN push notification sent:
  - Preparing: "[Restaurant] is preparing your order"
  - Walking: "Your order is on the way! ~X min walk"
  - Delivered: "Your order has been delivered. Enjoy!"
```

### 3. Content Specification

| Key | English | Arabic |
|---|---|---|
| notification_preparing | "{restaurant} is preparing your order" | "{restaurant} يحضّر طلبك" |
| notification_walking | "Your order is on the way! ~{min} min walk" | "طلبك في الطريق! ~{min} دقيقة سيراً" |
| notification_delivered | "Your order has been delivered. Enjoy!" | "تم توصيل طلبك. بالعافية!" |
| notification_permission | "Enable notifications to get order updates" | "فعّل الإشعارات لمتابعة طلبك" |
| notification_denied | "Notifications blocked. Check your browser settings." | "الإشعارات محظورة. تحقق من إعدادات المتصفح." |

### 4. Wave Dependencies

**Depends On:** Wave 3.2.1 (Status changes trigger notifications); PWA service worker setup
**Exposes To:** No downstream.
**Implementation:** Web Push API; VAPID keys stored in Secrets Manager; notification service in walk-worker pod.

### 5. QO Testing Guidance

**Risk Areas:** iOS Safari PWA push support (limited; check current 2026 status); notification permission UX.
**Test Focus:** Verify on Android Chrome + iOS Safari; test denied permission flow; verify correct message per status.
**Approach:** Manual device testing; integration for notification service.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

