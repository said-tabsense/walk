# Wave 3.1.1: Analytics Aggregation Service

| Field | Value |
|---|---|
| **Feature** | Analytics Data Collection + Aggregation |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Merchants need visibility into workplace ordering performance but raw order data is not queryable for analytics.

**Outcome:** Background cron service aggregates order data hourly by organization, time period, and item popularity. Data served via API for dashboard consumption.

**Metrics:**
- Aggregation runs hourly via `walk-cron` pod
- Data accurate within 1 hour of order completion
- Analytics query response < 500ms
- Cancelled orders excluded from revenue metrics

### 2. API Contracts

**GET /api/v1/merchants/:id/walk-analytics**
```
Auth: Merchant JWT
Query: ?period=7d|30d|90d&org_id=uuid(optional)
Response 200: {
  "period": "7d",
  "summary": { "total_orders", "total_revenue_sar", "avg_order_value_sar", "group_order_pct", "unique_customers", "repeat_order_rate" },
  "by_organization": [{ "org_id", "org_name", "orders", "revenue_sar", "avg_order_value_sar", "top_items" }],
  "daily_trend": [{ "date", "orders", "revenue_sar" }]
}
```

### 3. Content Specification

| Key | English | Arabic |
|---|---|---|
| total_orders | "Total Orders" | "إجمالي الطلبات" |
| total_revenue | "Total Revenue" | "إجمالي الإيرادات" |
| avg_order | "Avg. Order Value" | "متوسط قيمة الطلب" |
| group_pct | "Group Orders" | "الطلبات الجماعية" |
| unique_customers | "Unique Customers" | "عملاء فريدون" |
| repeat_rate | "Repeat Rate" | "معدل التكرار" |
| no_data | "No orders yet. Analytics will appear after your first Walk order." | "لا توجد طلبات بعد. ستظهر التحليلات بعد أول طلب Walk." |

### 4. Wave Dependencies

**Depends On:** Surge 1+2 order data (walk_orders table)
**Exposes To:** Wave 3.1.2 (Dashboard UI)
**Database:** `walk_analytics` materialized view or summary table; runs as `walk-cron` scheduled job
**Timezone:** All aggregation in Asia/Riyadh (UTC+3)

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

