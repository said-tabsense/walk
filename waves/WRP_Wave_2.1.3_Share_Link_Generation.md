# Wave 2.1.3: Share Link Generation

| Field | Value |
|---|---|
| **Feature** | Share Link Generation |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 1-2 hours |

### 1. Intent Definition

**Problem:** Organizers need to share the group order link with colleagues easily via WhatsApp, Slack, or clipboard.

**Outcome:** Share sheet with pre-formatted messages for WhatsApp and Slack, plus copy-to-clipboard. Uses Web Share API where available.

### 2. User Stories

**US-1: Share via WhatsApp**
```
GIVEN organizer taps "Share" and selects WhatsApp
WHEN WhatsApp opens
THEN pre-formatted message: "Group order from [Restaurant]! Join and add your items: [link]. Order closes in [time remaining]."
```

**US-2: Copy Link**
```
GIVEN organizer taps "Copy Link"
WHEN clipboard updated
THEN toast: "Link copied!"
```

### 3. Content Specification

| Key | English | Arabic |
|---|---|---|
| share_whatsapp_msg | "Group order from {restaurant}! Join and add your items: {link}. Order closes in {time_remaining}." | "طلب جماعي من {restaurant}! انضم وأضف عناصرك: {link}. يُغلق الطلب خلال {time_remaining}." |
| link_copied | "Link copied!" | "تم نسخ الرابط!" |
| share_title | "Share Group Order" | "مشاركة الطلب الجماعي" |
| whatsapp_btn | "WhatsApp" | "واتساب" |
| slack_btn | "Slack" | "سلاك" |
| copy_link_btn | "Copy Link" | "نسخ الرابط" |

### 4. Wave Dependencies

**Depends On:** Wave 2.1.1 (share link URL from session)
**Exposes To:** No downstream; utility Wave.
**Implementation:** Web Share API with fallback; WhatsApp deep link: `https://wa.me/?text={encoded}`

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

