# Business Requirement Document (BRD)
## Retail Inventory Analytics Dashboard – CircuitHub Electronics

---

### 1. Client Background

CircuitHub Electronics is a multi-warehouse electronics retailer (US-based) selling laptops, smartphones, home appliances, and accessories across 4 regional warehouses. Inventory data currently lives in **separate Excel/CSV exports per warehouse**, generated weekly from each warehouse's local stock system — there is no centralized inventory database.

### 2. Why CircuitHub Engaged a Consulting Partner (Afzalytics)

CircuitHub's internal Operations team had been managing inventory reporting manually, but hit a point where the manual process was actively costing the business money and decision-making speed:

- **No single source of truth.** Each warehouse exports its own file, with slightly different column names and formats (e.g., "SKU_Code" vs "Product_ID"), making consolidated reporting error-prone and slow.
- **Reporting took 3–4 days per cycle.** By the time a consolidated view was manually built in Excel (VLOOKUPs, pivot tables), the data was already a week old.
- **Stock-outs and excess stock were happening simultaneously, invisibly.** A SKU could be out of stock in one warehouse while sitting unsold in another — nobody had a centralized view to catch this.
- **Numbers didn't match between people.** Different team members calculated "Days of Inventory on Hand" differently depending on whether they used a point-in-time snapshot or movement logs, leading to conflicting figures in meetings.
- **Slow-moving stock was caught too late.** Without systematic tracking, capital stayed tied up in aging inventory for months before anyone noticed.

CircuitHub did not need a new inventory system — they needed their **existing Excel/CSV exports turned into a reliable, automated, single source of analytical truth**, with clear, auditable business rules replacing ad-hoc spreadsheet logic. This is the engagement scope for Afzalytics.

### 3. Business Objective

Build a Power BI solution that consolidates multi-warehouse Excel/CSV inventory data into one model, applies consistent business-rule-driven KPIs, and gives Operations a self-service dashboard to spot stock-outs, excess stock, and slow-moving SKUs — without waiting on manual reporting cycles.

### 4. Stakeholders

| Role | Interest |
|---|---|
| Head of Operations | Cross-warehouse visibility, stock-out/excess-stock exposure, trusted single number for reporting |
| Warehouse Managers | Warehouse-level performance, SKU-level stock status for their location |
| Category Managers | Category/product-level slow-moving and aging trends |
| Afzalytics (BI Developer) | Translate inconsistent source files and informal business rules into a structured, validated model |

### 5. Scope

**In Scope:**
- Consolidation of weekly Excel/CSV inventory exports from 4 warehouses
- Stock on Hand, by warehouse / category / product / SKU
- Stock-out Rate and Excess-stock identification
- Slow-moving Inventory % and Inventory Aging
- Days of Inventory on Hand (DOH)
- SKU classification: Low-stock, Healthy-stock, Excess-stock, Slow-moving
- Warehouse performance comparison
- Inbound/outbound movement trend
- Drill-down/drill-through from summary to SKU-level detail

**Out of Scope (v1):**
- Automated file ingestion / scheduled refresh pipeline (manual file drop for v1)
- Supplier/procurement lead-time analysis
- Financial valuation of inventory (cost/margin layer)
- Demand forecasting

### 6. Data Sources (as received from client)

| File Type | Frequency | Contents | Known Issues |
|---|---|---|---|
| Warehouse Stock Snapshot (CSV export, 1 per warehouse) | Weekly | Point-in-time stock count per SKU per warehouse | Inconsistent column naming across warehouses, missing SKU codes, mixed date formats |
| Inventory Movement Log (CSV) | Daily | Stock in/out transactions per SKU per warehouse | Duplicate entries, some records missing warehouse code, inconsistent movement-type labels |
| Product Master (Excel) | As-needed updates | SKU, Product Name, Category, Sub-category, Brand | Occasional duplicate SKUs with inconsistent naming |

**Key modeling challenge flagged upfront:** Snapshot data and Movement data are at **different grains** (point-in-time balance vs. transactional event) and must not be blended naively — aggregating them together without the correct relationship design would silently produce incorrect Stock on Hand and DOH figures. This is addressed explicitly in the data model.

### 7. Key Metrics / Business Rules

| KPI | Definition / Rule |
|---|---|
| Stock on Hand | Latest available snapshot quantity per SKU per warehouse |
| Stock-out Rate | % of SKUs with Stock on Hand = 0, out of total active SKUs, per warehouse/category |
| Excess Stock | SKU where Stock on Hand > 60 days of average demand |
| Slow-moving Inventory % | % of SKU-warehouse positions with no outbound movement in the last 60 days |
| Inventory Aging | Days since last inbound stock receipt, per SKU |
| Days of Inventory on Hand (DOH) | Stock on Hand ÷ Average Daily Outbound Movement (trailing 30 days) |
| SKU Status Classification | Low-stock (DOH < 10 days) · Healthy (10–60 days) · Excess (> 60 days) · Slow-moving (no movement 60+ days) |

### 8. Reporting Cadence

- Weekly refresh (manual file drop for v1), aligned to the warehouse snapshot cycle
- Ad-hoc drill-through investigation by Warehouse Managers as needed

### 9. Report Pages

1. **Inventory Overview**: KPI cards (Stock on Hand, Stock-out Rate, Slow-moving %, Median DOH across SKUs), stock trend by warehouse, stock distribution by category
2. **Warehouse Performance**: Cross-warehouse comparison of stock-out rate, Days of Inventory on Hand, slow-moving % and inventory aging
3. **Inventory Movement**: Inbound vs outbound trend over time, movement volume by category
4. **Product / Category Analysis**: Category > SubCategory > SKU drill-down, SKU status mix, inventory aging vs stock on hand
5. **SKU Detail** (Drill-through from Warehouse or Category): Individual SKU status, Days of Inventory, aging and classification

### 10. Assumptions

- Each warehouse's weekly snapshot represents end-of-week stock position
- Movement log "Outbound" entries represent sales/dispatches; "Inbound" represents receipts
- A SKU with no snapshot record in a given week is assumed out of stock at that warehouse for that week
- End users are Operations/Warehouse staff — dashboard must support both summary and granular (SKU-level) views without requiring analyst assistance

---
*Prepared by: Mirza Afzal Baig | Afzalytics*
*Date: October 2026*
