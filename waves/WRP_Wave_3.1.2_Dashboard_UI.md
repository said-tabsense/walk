# Wave 3.1.2: Dashboard UI

| Field | Value |
|---|---|
| **Feature** | Merchant Workplace Analytics Dashboard |
| **Version** | 1.0 |
| **Product Owner** | Nidal Khalifeh |
| **Date** | April 18, 2026 |
| **Wave Duration** | 2-3 hours |

### 1. Intent Definition

**Problem:** Merchants need a visual dashboard to understand workplace ordering performance.

**Outcome:** Dashboard section in merchant portal: KPI cards, daily trend line chart, per-organization breakdown table.

**Metrics:**
- Dashboard loads < 2 seconds
- Data matches aggregation API output exactly
- Charts render for 1-day, 7-day, 30-day, 90-day ranges

### 2. Wireframe Description

**Location:** Merchant Dashboard → Walking Delivery → Analytics tab

**Components:**
1. **Period Selector:** Tabs: "7 Days" | "30 Days" | "90 Days"
2. **KPI Cards (4 across):** Total Orders | Revenue (SAR) | Avg Order Value | Repeat Rate. Each: large number + trend arrow (vs previous period)
3. **Daily Trend Chart:** Line chart (Recharts), x-axis dates, y-axis orders + revenue (dual axis). Tooltip on hover.
4. **Organization Breakdown Table:** Columns: Org Name | Orders | Revenue | Avg Order | Top Item. Sorted by revenue descending. Clickable rows for org detail (future).

### 3. Content Specification

| Key | English | Arabic |
|---|---|---|
| analytics_title | "Walk Analytics" | "تحليلات Walk" |
| period_7d | "7 Days" | "7 أيام" |
| period_30d | "30 Days" | "30 يوم" |
| period_90d | "90 Days" | "90 يوم" |
| vs_previous | "vs previous period" | "مقارنة بالفترة السابقة" |

### 4. Wave Dependencies

**Depends On:** Wave 3.1.1 (Analytics API)
**Exposes To:** No downstream; terminal UI Wave.

---

---

*End of Wave-Ready Package*
*Generated: April 18, 2026*

