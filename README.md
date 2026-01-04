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
**SQL code:** 
WITH t AS (
SELECT
Date,
LN(Average_selling_price) AS ln_price,
LN(Units_Ordered) AS ln_units
FROM seller_performance
WHERE Average_selling_price > 0 AND Units_Ordered > 0
),
stats AS (
SELECT
AVG(ln_price) AS avg_ln_p,
AVG(ln_units) AS avg_ln_u
FROM t
)
SELECT
-- slope = covariance(ln_price, ln_units) / variance(ln_price)
SUM((t.ln_price - stats.avg_ln_p) * (t.ln_units - stats.avg_ln_u))
/ NULLIF(SUM((t.ln_price - stats.avg_ln_p) * (t.ln_price - stats.avg_ln_p)), 0)
AS price_elasticity
FROM t
CROSS JOIN stats;

**Output:** 
<img width="778" height="199" alt="Screenshot (1)" src="https://github.com/user-attachments/assets/4884867a-7ff6-461c-b9b3-76fecd2efe1f" />


---

### **Insight B — Customer Intent Index (CII)**
**Problem:** Identify buying intent across channels.  
**Columns:** Units_Ordered, Page_views_*  
**Overall Averages:**  
- CII_total = **0.01266**  
- CII_mobile = **0.01661**  
- CII_browser = **0.05689**  
**Meaning:** Browser intent is 3.5× higher than mobile — fix mobile PDP.  
**SQL:** 
SELECT
AVG(Units_Ordered / NULLIF(Page_views_total, 0)) AS avg_CII_total,
AVG(Units_Ordered / NULLIF(Page_views_mobile_app, 0)) AS avg_CII_mobile,
AVG(Units_Ordered / NULLIF(Page_views_browser, 0)) AS avg_CII_browser
FROM seller_performance;

**Output:** <img width="778" height="199" alt="Screenshot (2)" src="https://github.com/user-attachments/assets/84f86814-fca4-4b9f-9a7e-5a5aa1ecf445" />


---

### **Insight C — Featured Offer (Buy Box) Impact**
**Problem:** Quantify revenue loss when Buy Box drops.  
**Columns:** Featured_offer_percentage, Ordered_Product_Sales  
**Result:**  
- Regression slope = **1.5874**  
- Strong correlation between Buy Box % & revenue  
**Meaning:** Losing even 1–2% Featured Offer % materially reduces sales.  
**SQL:** 
WITH f AS (
SELECT
Date,
Featured_offer_percentage,
Ordered_Product_Sales,
(Ordered_Product_Sales - LAG(Ordered_Product_Sales) OVER (ORDER BY Date)) / NULLIF(LAG(Ordered_Product_Sales) OVER (ORDER BY Date), 0) AS sales_pct_change,
(Featured_offer_percentage - LAG(Featured_offer_percentage) OVER (ORDER BY Date)) / NULLIF(LAG(Featured_offer_percentage) OVER (ORDER BY Date), 0) AS feat_pct_change,
(Featured_offer_percentage - LAG(Featured_offer_percentage) OVER (ORDER BY Date)) AS feat_diff_pp,
(Ordered_Product_Sales - LAG(Ordered_Product_Sales) OVER (ORDER BY Date)) AS sales_diff_abs
FROM seller_performance
)
SELECT
SUM((feat_pct_change - AVG_feat) * (sales_pct_change - AVG_sales)) / NULLIF(SUM((feat_pct_change - AVG_feat) * (feat_pct_change - AVG_feat)),0) AS slope_pct_change
FROM (
SELECT feat_pct_change, sales_pct_change,
(SELECT AVG(feat_pct_change) FROM f WHERE feat_pct_change IS NOT NULL AND sales_pct_change IS NOT NULL) AS AVG_feat,
(SELECT AVG(sales_pct_change) FROM f WHERE feat_pct_change IS NOT NULL AND sales_pct_change IS NOT NULL) AS AVG_sales
FROM f
WHERE feat_pct_change IS NOT NULL AND sales_pct_change IS NOT NULL
) x;
  
**Output:** <img width="778" height="301" alt="Screenshot (3)" src="https://github.com/user-attachments/assets/f9b8871f-a587-49b8-b4ec-0156c10f60a1" />


---

### **Insight D — Refund Pressure Score (RPS)**
**Problem:** Identify days where refund risk was damaging.  
**Columns:** Refund_rate, Units_refunded, Units_Ordered  
**Top Spike Days:**  
- 2024-10-17 — RPS ~ 281  
- 2024-09-15 — RPS ~ 160  
- 2024-06-21 — RPS ~ 62  
**Meaning:** Likely SKU-level or fulfillment issues — requires investigation.  
**SQL:** 
SELECT
Date,
Refund_rate,
Units_refunded,
Units_Ordered,
(Refund_rate * 0.6) + ((Units_refunded / NULLIF(Units_Ordered, 0)) * 0.4) AS RPS
FROM seller_performance
ORDER BY RPS DESC LIMIT 3;

**Output:** <img width="778" height="205" alt="Screenshot (4)" src="https://github.com/user-attachments/assets/e22724b8-0a36-4eeb-ab18-bf85e1492a4f" />


---

### **Insight E — Conversion Funnel**
**pv → session:** 0.7447  
**session → order item:** 0.0161  
**order item → unit:** 1.0569  
**Meaning:** Major drop at the session → order stage. Customers visit but do not buy — PDP improvements needed.  
**SQL:**
SELECT
AVG(Sessions_total / NULLIF(Page_views_total, 0)) AS avg_pv_to_session,
AVG(Total_Order_Items / NULLIF(Sessions_total, 0)) AS avg_session_to_orderitem,
AVG(Units_Ordered / NULLIF(Total_Order_Items, 0)) AS avg_orderitem_to_unit
FROM seller_performance;  

**Output:** <img width="794" height="188" alt="Screenshot (5)" src="https://github.com/user-attachments/assets/f1a7c0eb-4b58-4c28-a148-eec0b48fb011" />


---

### **Insight F — High-Traffic Low-Sales Anomalies**
**Problem:** Traffic increases but sales collapse on certain days.  
**Detected Anomaly Dates:**  
- 2024-05-16  
- 2025-01-20  
- 2025-01-27  
- 2025-01-28  
**Meaning:** Possible Buy Box loss, suppression, or technical issues.  
**SQL:** 
WITH stats AS (
SELECT AVG(Ordered_Product_Sales) AS avg_sales, AVG(Page_views_total) AS avg_pv FROM seller_performance
)
SELECT s.Date, s.Ordered_Product_Sales, s.Page_views_total
FROM seller_performance s
CROSS JOIN stats
WHERE s.Ordered_Product_Sales < 0.6 * stats.avg_sales
AND s.Page_views_total > stats.avg_pv
ORDER BY s.Date LIMIT 4;

**Output:** <img width="786" height="252" alt="Screenshot (6)" src="https://github.com/user-attachments/assets/baa86dc0-3301-4ff5-8598-ed757e0b6554" />


---

### **Insight G — Revenue Per Session**
**Avg Revenue/Session:** **₹9.80**  
**Top Days:**  
- May 6: ₹20.28  
- Aug 11: ₹19.74  
- May 8: ₹19.14  
**Meaning:** Some days are extremely efficient — replicate those conditions.  
**SQL:** 
SELECT Date, Ordered_Product_Sales / NULLIF(Sessions_total, 0) AS revenue_per_session
FROM seller_performance
ORDER BY revenue_per_session DESC
LIMIT 3;

**Output:** <img width="772" height="170" alt="Screenshot (7)" src="https://github.com/user-attachments/assets/18fc2ca4-0d56-4c86-addf-6fcba530d30c" />

---

### **Insight H — Offer Count Saturation**
**Problem:** How competition affects conversion.  
**Finding:**  
- Best conversion at Offer Count ≈ **1287**  
**Meaning:** Conversion peaks at certain competition levels — not a linear decline.  
**SQL:** 
WITH offer_stats AS (
SELECT Average_offer_count, AVG(Total_Order_Items / NULLIF(Sessions_total, 0)) AS avg_conversion
FROM seller_performance
GROUP BY Average_offer_count
)
SELECT Average_offer_count, avg_conversion
FROM offer_stats
ORDER BY avg_conversion DESC
LIMIT 1;

**Output:** <img width="790" height="180" alt="Screenshot (8)" src="https://github.com/user-attachments/assets/cf6be32a-c070-495e-ae7e-0a5a96b14865" />


---


## 8) Conclusion
The analysis reveals that price, Buy Box stability, refund spikes, and mobile PDP performance are the strongest drivers of business outcomes. Addressing these areas will improve sales consistency, reduce losses, and increase conversion across channels.

---

## 9) Next Steps
- Add SKU-level analytics for deeper operational insights.  
- Build a dashboard (Tableau, Power BI, or Streamlit).  
- Automate SQL → CSV → dashboard refresh.  
- Add Python ML models (forecasting, anomaly detection).  
- Expand analysis to advertising, inventory, and competition data.  

---

## 10) References
- Amazon Seller Central documentation  
- “Data Science for Business” — Provost & Fawcett  
- SQL window functions documentation  


