# 📐 Enterprise DAX Measures (`_Measures` Table)

All calculations in the **Global Supply Chain Control Tower** are stored in a dedicated `_Measures` table following enterprise Kimball Star Schema standards.

---

## 📊 Page 1: Logistics & Carrier SLA Health Measures

```dax
-- 1. Total Shipped Orders (Excludes Canceled & Fraudulent Orders)
Total Shipped Orders = 
CALCULATE(
    DISTINCTCOUNT(silver_fact_order_items[order_id]),
    NOT(silver_fact_order_items[order_status] IN {"CANCELED", "SUSPECTED_FRAUD"})
)

-- 2. On Time Orders (Zero Late Delivery Risk)
On Time Orders = 
CALCULATE(
    DISTINCTCOUNT(silver_fact_order_items[order_id]),
    NOT(silver_fact_order_items[order_status] IN {"CANCELED", "SUSPECTED_FRAUD"}),
    silver_fact_order_items[late_delivery_risk] = 0
)

-- 3. Average On-Time In-Full (OTIF) Rate %
Avg OTIF Rate % = 
DIVIDE([On Time Orders], [Total Shipped Orders], 0) * 100

-- 4. Average Carrier SLA Breach Rate %
Avg SLA Breach Rate % = 
DIVIDE([Total Shipped Orders] - [On Time Orders], [Total Shipped Orders], 0) * 100

-- 5. Net Revenue Exposed to SLA Breach ($)
Revenue at SLA Risk = 
CALCULATE(
    SUM(silver_fact_order_items[net_sales]),
    NOT(silver_fact_order_items[order_status] IN {"CANCELED", "SUSPECTED_FRAUD"}),
    silver_fact_order_items[late_delivery_risk] = 1
)

-- 6. Scheduled / Promised Lead Time (Days)
Scheduled Lead Time (Days) = 
AVERAGE(silver_fact_order_items[scheduled_lead_time_days])

-- 7. Actual Fulfilled Lead Time (Days)
Actual Lead Time (Days) = 
AVERAGE(silver_fact_order_items[actual_lead_time_days])

-- 8. Average Lead Time Delay Variance (Days)
Avg Lead Time Delay (Days) = 
AVERAGE(silver_fact_order_items[lead_time_variance_days])

-- 9. OTIF Variance vs. 95% Corporate SLA Target
OTIF Gap vs 95% Target = 
[Avg OTIF Rate %] - 95.0

-- 10. Net Profit Exposed to SLA Breach ($)
Profit at SLA Risk = 
CALCULATE(
    SUM(silver_fact_order_items[order_item_profit]),
    NOT(silver_fact_order_items[order_status] IN {"CANCELED", "SUSPECTED_FRAUD"}),
    silver_fact_order_items[late_delivery_risk] = 1
)
```

---

## 📦 Page 2: Prescriptive Inventory & ROP Optimization Measures

```dax
-- 11. Total SKUs Requiring Immediate Reorder (Below ROP or Safety Stock)
SKUs Needing Reorder = 
CALCULATE(
    DISTINCTCOUNT(gold_sku_rop_optimization[product_id]),
    gold_sku_rop_optimization[inventory_status] <> "HEALTHY: Sufficient Stock"
)

-- 12. Flagship Class A SKUs at Stockout Risk
Class A Alerts = 
CALCULATE(
    DISTINCTCOUNT(gold_sku_rop_optimization[product_id]),
    gold_sku_rop_optimization[inventory_status] <> "HEALTHY: Sufficient Stock",
    LEFT(gold_sku_rop_optimization[abc_class], 7) = "Class A"
)

-- 13. 30-Day ML Forecasted Demand (Units)
30D Forecast (Units) = 
SUM(gold_sku_rop_optimization[forecast_30d_demand])

-- 14. Working Capital Tied Up in Safety Stock Buffers ($)
Safety Stock ($) = 
SUM(gold_sku_rop_optimization[safety_stock_value_usd])

-- 15. Working Capital Required for Recommended Purchase Orders ($)
PO Capital Needed ($) = 
SUM(gold_sku_rop_optimization[recommended_po_value_usd])

-- 16. Total Historical Net Sales ($) (Used for ABC Pareto Curve)
Total Net Sales = 
SUM(silver_fact_order_items[net_sales])

-- 17. Total Unique Catalog SKUs
Total SKU Count = 
DISTINCTCOUNT(gold_sku_rop_optimization[product_id])
```
