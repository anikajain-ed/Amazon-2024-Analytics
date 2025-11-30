# Amazon Seller Performance Analytics

## Index
1. Project Overview  
2. Goals (Business-Focused)  
3. Data Description & Confidentiality  
4. Methodology  
5. ERD & Data Model  
6. Indexing & Performance Notes  
7. Insights  
   - Insight A: Price Elasticity  
   - Insight B: Customer Intent Index  
   - Insight C: Featured Offer Impact  
   - Insight D: Refund Pressure Score  
   - Insight E: Conversion Funnel  
   - Insight F: High-Traffic Low-Sales Anomalies  
   - Insight G: Revenue Per Session  
   - Insight H: Offer Count Saturation  
8. How Code & Outputs Are Organized  
9. Conclusion  
10. Next Steps  
11. References  

---

## 1) Project Overview
This project analyzes one year of Amazon seller performance data (daily aggregated metrics) to uncover business-critical insights. Using SQL-based analytics, the project identifies operational inefficiencies, conversion bottlenecks, Buy Box instability patterns, refund spikes, and customer behavior trends.
The goal is to generate *data-driven insights* that help improve sales, reduce operational losses, optimize pricing, and strengthen listing performance.

---

## 2) Goals (Business-Focused)
- Identify key factors influencing sales performance.  
- Quantify the effect of Buy Box (Featured Offer %) on revenue.  
- Detect refund spikes and operational problems.  
- Evaluate user behavior across mobile vs browser channels.  
- Improve overall conversion rates with data-backed recommendations.  
- Build a structured analytics workflow for long-term decision-making.  

---

## 3) Data Description & Confidentiality
**Source:** Internal Amazon Seller Central performance exports.  
**Data Type:** Aggregated daily metrics only — *no SKU-level or confidential customer data is uploaded*.

This repository **does NOT contain raw company data**.  
All uploaded outputs are **anonymized, masked, or sample-only**.

### Column Overview
- **Sales Metrics:** Ordered_Product_Sales, Shipped_product_sales, Units_Ordered, Orders_shipped  
- **Conversion Metrics:** Total_Order_Items, Order_item_session_percentage  
- **Price Metrics:** Average_selling_price, Average_sales_per_order_item  
- **Traffic Metrics:** Page_views_total, Page_views_browser, Page_views_mobile_app, Sessions_total  
- **Buy Box Metrics:** Featured_offer_percentage  
- **Quality Metrics:** Units_refunded, Refund_rate, Negative_feedback_received  
- **Competition Metrics:** Average_offer_count  
- **Claims & Support:** A_to_z_claims_granted, Claims_amount  

> Confidential raw files are never included. Only derived CSVs containing aggregated numbers are uploaded.

---

## 4) Methodology
- Load cleaned Amazon export into SQL table (`seller_performance`).
- Use SQL window functions, joins, and aggregations to compute:
  - Price Elasticity  
  - Featured Offer Sensitivity  
  - Conversion Funnel  
  - Refund Pressure Score (RPS)  
  - Customer Intent Index  
  - Anomaly Detection  
  - Revenue Per Session  
- Export SQL outputs to `/outputs/`.
- Document each insight clearly with SQL + explanation.

Tools used:
- MySQL Workbench  
- Python (optional, for supplemental charts)  
- GitHub for reproducibility  

---

## 5) ERD & Data Model
**ERD File:** `erd/seller_performance_erd.png`  
- Single main table: `seller_performance` keyed by `Date`.  
- Future enhancement: SKU-level table, returns table, claims table.

---

## 6) Indexing & Performance Notes
Recommended indexes for scaling:
```
CREATE INDEX idx_date ON seller_performance(Date);
CREATE INDEX idx_featured ON seller_performance(Featured_offer_percentage);
```

If SKU is added:
```
CREATE INDEX idx_sku_date ON seller_performance(sku, Date);
```

---

## 7) Insights

### **Insight A — Price Elasticity of Demand**
**Problem:** Understand how sensitive customers are to price changes.  
**Columns Used:** Average_selling_price, Units_Ordered  
**Result:** Elasticity = **-1.89237**  
**Meaning:** A 1% increase in price leads to ~1.89% drop in units ordered.  
**Business Impact:** Price increases should be cautious; promotions strongly influence demand.  
**SQL File:** `code/sql/price_elasticity.sql`  
**Output File:** `outputs/price_elasticity.csv`

---

### **Insight B — Customer Intent Index (CII)**
**Problem:** Identify buying intent across channels.  
**Columns:** Units_Ordered, Page_views_*  
**Overall Averages:**  
- CII_total = **0.01266**  
- CII_mobile = **0.01661**  
- CII_browser = **0.05689**  
**Meaning:** Browser intent is 3.5× higher than mobile — fix mobile PDP.  
**SQL:** `code/sql/customer_intent_index.sql`  
**Output:** `outputs/cii_overall.csv`

---

### **Insight C — Featured Offer (Buy Box) Impact**
**Problem:** Quantify revenue loss when Buy Box drops.  
**Columns:** Featured_offer_percentage, Ordered_Product_Sales  
**Result:**  
- Regression slope = **1.5874**  
- Strong correlation between Buy Box % & revenue  
**Meaning:** Losing even 1–2% Featured Offer % materially reduces sales.  
**SQL:** `code/sql/featured_offer_impact.sql`  
**Output:** `outputs/featured_offer_impact.csv`

---

### **Insight D — Refund Pressure Score (RPS)**
**Problem:** Identify days where refund risk was damaging.  
**Columns:** Refund_rate, Units_refunded, Units_Ordered  
**Top Spike Days:**  
- 2024-10-17 — RPS ~ 281  
- 2024-09-15 — RPS ~ 160  
- 2024-06-21 — RPS ~ 62  
**Meaning:** Likely SKU-level or fulfillment issues — requires investigation.  
**SQL:** `code/sql/refund_pressure.sql`  
**Output:** `outputs/rps_summary.csv`

---

### **Insight E — Conversion Funnel**
**pv → session:** 0.7447  
**session → order item:** 0.0161  
**order item → unit:** 1.0569  
**Meaning:** Major drop at the session → order stage. Customers visit but do not buy — PDP improvements needed.  
**SQL:** `code/sql/funnel_rates.sql`  
**Output:** `outputs/funnel_per_day.csv`

---

### **Insight F — High-Traffic Low-Sales Anomalies**
**Problem:** Traffic increases but sales collapse on certain days.  
**Detected Anomaly Dates:**  
- 2024-05-16  
- 2025-01-20  
- 2025-01-27  
- 2025-01-28  
**Meaning:** Possible Buy Box loss, suppression, or technical issues.  
**SQL:** `code/sql/anomaly_detection.sql`  
**Output:** `outputs/anomalies.csv`

---

### **Insight G — Revenue Per Session**
**Avg Revenue/Session:** **₹9.80**  
**Top Days:**  
- May 6: ₹20.28  
- Aug 11: ₹19.74  
- May 8: ₹19.14  
**Meaning:** Some days are extremely efficient — replicate those conditions.  
**SQL:** `code/sql/revenue_per_session.sql`  
**Output:** `outputs/revenue_per_session.csv`

---

### **Insight H — Offer Count Saturation**
**Problem:** How competition affects conversion.  
**Finding:**  
- Best conversion at Offer Count ≈ **1287**  
**Meaning:** Conversion peaks at certain competition levels — not a linear decline.  
**SQL:** `code/sql/offer_count_saturation.sql`  
**Output:** `outputs/offer_count_buckets.csv`

---

## 8) How Code & Outputs Are Organized
```
/code/sql/                → SQL files for each insight  
/outputs/                 → CSV files for each insight  
/visuals/                 → charts (optional)
/erd/                     → ERD diagram  
/data/sample/             → optional anonymized sample dataset  
```

---

## 9) Conclusion
The analysis reveals that price, Buy Box stability, refund spikes, and mobile PDP performance are the strongest drivers of business outcomes. Addressing these areas will improve sales consistency, reduce losses, and increase conversion across channels.

---

## 10) Next Steps
- Add SKU-level analytics for deeper operational insights.  
- Build a dashboard (Tableau, Power BI, or Streamlit).  
- Automate SQL → CSV → dashboard refresh.  
- Add Python ML models (forecasting, anomaly detection).  
- Expand analysis to advertising, inventory, and competition data.  

---

## 11) References
- Amazon Seller Central documentation  
- “Data Science for Business” — Provost & Fawcett  
- SQL window functions documentation  


