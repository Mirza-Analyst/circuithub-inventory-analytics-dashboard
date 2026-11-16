# Project Blueprint
## CircuitHub Electronics: Retail Inventory Analytics Dashboard

**Developer:** Mirza Afzal Baig | Afzalytics
**Tool Stack:** Power Query, Power BI (DAX), Excel/CSV sources
**Domain:** Retail (Electronics, 4 regional warehouses, US)

---

## 1. Business Context

CircuitHub manages inventory across 4 warehouses (Columbus, Denver, Memphis, Newark) using separate weekly Excel/CSV exports per warehouse, with no single source of truth. Manual consolidation took 3 to 4 days per cycle, stock-outs and excess stock occurred simultaneously across locations without anyone noticing, and Days of Inventory was calculated differently by different people.

**Objective:** One consolidated model with consistent, rule-based KPIs and a self-service dashboard for Operations.

Full requirement document: *CircuitHub_Inventory_Dashboard_Requirement_Doc*.

---

## 2. Data Sources (6 files: 1 Excel, 5 CSV)

| File | Grain | Notes |
|---|---|---|
| `Product_Master.xlsx` (Excel) | 1 row per SKU | 89 SKUs, 10 duplicate SKU entries with inconsistent naming |
| `WH_Columbus / Denver / Memphis / Newark_Stock_Snapshot.csv` (4 files) | SKU x Week (point-in-time stock) | Each warehouse uses different column names and date formats; ~1% blank SKU codes; duplicate rows |
| `Inventory_Movement_Log.csv` | Transaction (daily Inbound/Outbound) | 3 mixed date formats, 10 spellings of movement type, warehouse code variants, blank SKU/warehouse, duplicates |

Snapshot and Movement data are at **different grains**, which drives the model design.

---

## 3. Power Query

**Snapshot files (each of 4):** set types (Date, Whole Number) with explicit locale where needed, remove rows with empty SKU, rename to a common schema (`SKU`, `Quantity`, `SnapshotDate`), add a `WarehouseName` column (the source file name is lost after append, so it is added before).
**Append:** 4 queries stacked into `Fact_Snapshot`, then Remove Duplicates (9,212 to 9,164 rows). The 4 individual queries are load-disabled (staging only).

**Movement_Log:**
- `MovementType`: 10 variants standardized to Inbound / Outbound
- `TransactionDate`: mixed formats parsed with `Date.FromText([TransactionDate], "en-US")`
- `WarehouseCode`: trim, variants standardized, blanks set to "Unknown" (kept, not dropped)
- `SKU`: trim, uppercase, empty removed
- Remove Duplicates across all columns (same transaction recorded twice), applied on the raw values before warehouse-name standardization so that genuine separate transactions are not merged (16,691 rows remain, 326 with Unknown warehouse)

**Dimensions:** `Dim_Product` (Product_Master, SKU deduplicated, 89 rows), `Dim_Warehouse` (4 unique warehouses, derived from Fact_Snapshot).

---

## 4. Data Model

```
                  Dim_Product
                 /     |      \
        Fact_Snapshot  |   Movement_Log
                 \     |      /
                 Dim_Warehouse      DateTable (to both facts)

   SKU_Status_Snapshot (calculated, SKU x Warehouse) -> Dim_Product, Dim_Warehouse
```

Two fact tables at different grains (stock snapshot vs movement transaction), sharing Product, Warehouse and Date dimensions, with no direct relationship between the two facts (Galaxy / fact-constellation pattern).

**DateTable:** DAX CALENDAR for 2024, marked as Date Table, `YearMonth` sorted by `YearMonthSort`.

**SKU_Status_Snapshot:** calculated table at SKU x Warehouse grain holding `Status`, `Aging Days`, `Aging Bucket`, so status can be used as a slicer and counted. Built at SKU x Warehouse (not SKU only) because blending all warehouses per SKU hides local low-stock situations.

---

## 5. Key DAX Measures

```dax
Stock on Hand = 
VAR LatestSnapshotDate = CALCULATE(MAX(Fact_Snapshot[SnapshotDate]), ALLSELECTED(Fact_Snapshot))
RETURN CALCULATE(SUM(Fact_Snapshot[Quantity]), Fact_Snapshot[SnapshotDate] = LatestSnapshotDate)

Stock on Hand (Trend) = SUM(Fact_Snapshot[Quantity])

Stock-out Rate = DIVIDE([Stocked-out SKUs], [Total SKUs])

Avg Daily Outbound (30d) = 
VAR MaxDate = CALCULATE(MAX(Movement_Log[TransactionDate]), ALLSELECTED(Movement_Log))
RETURN DIVIDE(
    CALCULATE(SUM(Movement_Log[Quantity]), Movement_Log[MovementType] = "Outbound",
              Movement_Log[TransactionDate] >= MaxDate - 30, Movement_Log[TransactionDate] <= MaxDate), 30)

Days of Inventory on Hand = DIVIDE([Stock on Hand], [Avg Daily Outbound (30d)])

Slow-Moving Flag = IF(ISBLANK([Last Outbound Date]) || [Days Since Last Outbound] > 60, "Slow-Moving", "Active")

SKU Status = 
VAR DOH = [Days of Inventory on Hand]
RETURN SWITCH(TRUE(),
    [Slow-Moving Flag] = "Slow-Moving", "Slow-Moving",
    ISBLANK(DOH) && [Stock on Hand] = 0, "Low-Stock",
    ISBLANK(DOH), "Excess",      -- BLANK < 10 is TRUE in DAX, so blanks are handled explicitly
    DOH < 10, "Low-Stock",
    DOH <= 60, "Healthy",
    "Excess")

Slow-Moving % = DIVIDE(
    CALCULATE(COUNTROWS(SKU_Status_Snapshot), SKU_Status_Snapshot[Status] = "Slow-Moving"),
    COUNTROWS(SKU_Status_Snapshot))

Median DOH Across SKUs = MEDIANX(VALUES(Dim_Product[SKU]), [Days of Inventory on Hand])

Inventory Aging (Days) = DATEDIFF([Last Inbound Date],
    CALCULATE(MAX(Movement_Log[TransactionDate]), ALLSELECTED(Movement_Log)), DAY)
```

Supporting measures: Total SKUs, Stocked-out SKUs, Total Inbound/Outbound Units, Total Movement Volume, Last Outbound/Inbound Date, Days Since Last Outbound, Total SKU-Warehouse Records, Avg Aging (Days).

---

## 6. Report Pages (5)

| Page | Visuals |
|---|---|
| **1. Inventory Overview** | 4 KPI cards (Stock on Hand, Stock-out Rate, Slow-Moving %, Median DOH across SKUs) · Stock on Hand Trend by Warehouse (line) · Stock Distribution by Category (treemap) · Slicers: Date, Warehouse, Category, SKU |
| **2. Warehouse Performance** | Stock-out Rate by Warehouse (column) · Days of Inventory on Hand by Warehouse (bar) · Warehouse Performance Summary (table with conditional formatting on stock-out rate, days of inventory and slow-moving %) |
| **3. Inventory Movement** | Inbound vs Outbound Movement Trend (area) · Movement Volume by Category (column) · Movement Summary (multi-value card: inbound, outbound, records without warehouse) |
| **4. Product Category Analysis** | Category > SubCategory > SKU drill-down matrix · SKU Status Distribution (donut) · Inventory Aging vs Stock on Hand by SKU (scatter) · Status slicer (filters the status donut only, synced to Page 5) |
| **5. SKU Detail (drill-through)** | SKU-level table with Status conditional formatting. Drill-through fields: Warehouse (from Page 2) and Category (from Page 4) |

---

## 7. Validation

Key figures recomputed independently from the source CSVs (after the same cleaning logic):

| Metric | Source | Power BI |
|---|---|---|
| Stock on Hand (total) | 150,448 | 150,448 |
| By warehouse | 37,880 / 16,683 / 2,715 / 93,170 | identical |
| Stock-out Rate by warehouse | 1.1% / 8.0% / 18.4% / 1.1% | identical (rounded) |
| Days of Inventory on Hand (overall) | 70.84 | 70.72 (within 0.2%) |
| Days of Inventory by warehouse | 58.03 / 47.18 / 20.55 / 101.56 | 57.93 / 47.18 / 20.46 / 101.35 |
| Slow-Moving % (overall) | 0.24 | 0.24 (warehouse level within 1 SKU) |
| Unknown-warehouse movement rows | 326 | 326 |
| SKU status mix after blank-DOH fix | Low 15% / Slow 24% / Excess 28% / Healthy 32% | same |

---

## 8. Business Findings the Dashboard Surfaces

- **Memphis:** chronic stock-outs (about 18%) with only about 20 days of inventory, under-replenished.
- **Newark:** about 1% stock-out but about 100+ days of inventory and 62% of all stock, capital tied up in excess.
- **Home Appliances:** largest slow-moving share and lowest movement volume.
- Same SKU can be Low-Stock in one warehouse and Excess in another, which was exactly the client's original blind spot.

---

## 9. Key Design Decisions

- Snapshot vs Movement kept as separate facts, never blended in one table.
- "Latest snapshot only" logic for Stock on Hand; a separate plain-sum measure for trend visuals.
- SKU Status computed at SKU x Warehouse grain.
- Unknown warehouse records kept and labelled, not deleted.
- Dead-stock behaviour defined per SKU x Warehouse, so slow-moving varies by location.
- SKU-level DOH KPI uses the median (about 61 days), not the mean (about 160), because about 12% of SKUs have cover above one year and skew the mean; the overall ratio (about 71 days) is shown in the warehouse summary.
