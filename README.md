Customer Shopping Trends — India | Power BI Analysis

Power BI project analyzing Indian retail customer segments using star schema modeling and DAX

**Business Question

**> **Which customer segments and cities are driving the most revenue for the Indian retail business, and where is the company losing money through discounts and returns?**
This project analyzes 10,000 transactions from an Indian e-commerce/retail dataset to identify high-value customer segments, top-performing cities, and areas of revenue leakage — the kind of analysis a retail or consulting team would run before a marketing budget or inventory decision.

---

Dataset

Source: Customer Shopping Trends (India) — Kaggle
Size: 10,000 transactions, 26 raw columns
Coverage: Customer demographics, product details, purchase behavior, delivery, payment, returns, and subscription data across 20 Indian cities
Unique customers: 2,581 (average ~3.9 purchases per customer — genuine repeat-purchase behavior exists)
---

Tools Used
Power Query (M language) — data cleaning and transformation
Power BI Desktop — data modeling, DAX, dashboard visuals
DAX — measures and calculated columns
---

Step 1: Data Cleaning (Power Query)
Raw data wasn't "messy" in the traditional sense — most blanks were logical nulls (meaning "not applicable"), not missing data. Cleaning focused on making that meaning explicit rather than deleting or ignoring it.
Column	Issue	Fix
`Festival/Sale`	56% blank (not a sale purchase)	Conditional column → `"Regular Price"`
`Delivery Speed`	22% blank (offline orders have no delivery)	Conditional column → `"N/A"`
`Size`	~9% blank (accessories have no size)	Conditional column → `"N/A"` (`Size_Clean`)
Data types	Several numeric/date columns auto-detected as Text	Manually corrected: `Purchase Date` → Date, currency/decimal fields → Decimal Number, counts → Whole Number
Text columns	Invisible extra spaces/characters	`Trim` + `Clean` applied across all text columns
Key lesson applied: blank cells from a CSV import often come through as empty string `""`, not a true `null` — so conditions had to check for both (`[Column] = null or [Column] = ""`).
---

Star schema: `dim_customer` and `dim_product` connected to `fact_transactions`, linked via `Customer ID` and the composite `Product Key`.

**Step 2: Data Modeling — Star Schema
**The flat table was split into a proper star schema for clean relationships and better DAX performance:
`dim_customer` (2,581 rows) — Customer ID, Age, Gender, Location, Subscription Status, Frequency of Purchases
`dim_product` (6,230 rows) — Category, Item Purchased, Brand, Color, Size
`fact_transactions` (10,000 rows) — every transaction-level fact: amounts, quantities, discounts, delivery, payment, returns
Relationship challenge: `dim_product` had no single natural ID — a product is only unique as the combination of Category, Item, Brand, Color, and Size. Solved by building a composite key:
```

ProductKey = [Category] & "|" & [Item Purchased] & "|" & [Brand] & "|" & [Color] & "|" & [Size_Clean]
```

Built identically in both `dim_product` and `fact_transactions`, then used as the relationship key — the same concept as a composite primary key in SQL, replicated in Power Query since the source data didn't provide one.
Modeling rule applied throughout: if a value describes who the customer is (stable identity trait) → dimension table. If it describes what happened in a specific transaction (an outcome, a count, a snapshot-in-time value) → fact table. This is why `Return Status` and `Previous Purchases` were kept in `fact_transactions` rather than `dim_customer`, even though they relate to a customer.
---

Step 3: Calculated Columns
Business-logic segmentation was added directly on `dim_customer`, using `SWITCH(TRUE(), ...)` pattern:
```dax
Age Group = 
SWITCH(
    TRUE(),
    dim_customer[Age] < 25, "Under 25",
    dim_customer[Age] < 35, "25-34",
    dim_customer[Age] < 50, "35-49",
    "50+"
)

Frequency Segment = 
SWITCH(
    TRUE(),
    dim_customer[Frequency of Purchases] IN {"Weekly", "Fortnightly"}, "High Frequency",
    "Lower Frequency"
)
```

Why Calculated Columns, not Measures: both values describe a fixed property of one customer's row (a classification), not an aggregation across many rows. This matters practically — only calculated columns can be used cleanly in slicers, filters, and Top N logic; a measure version (built using `SELECTEDVALUE`) only resolves correctly at row-level context and breaks at totals/aggregated views.
---

Step 4: DAX Measures
```dax
Total Revenue = SUM(fact_transactions[Purchase Amount])

Total Orders = DISTINCTCOUNT(fact_transactions[Transaction ID])

Average Order Value = DIVIDE([Total Revenue], [Total Orders])

Total Customers = DISTINCTCOUNT(fact_transactions[Customer ID])
```

`DIVIDE()` used instead of `/` throughout — handles division-by-zero gracefully (returns blank instead of an error), standard DAX best practice.
---

Step 5: Analysis — High-Value Customer Segments
Question: Among subscribed (`Subscription Status = Yes`), high-frequency (`Weekly`/`Fortnightly` buyers), which cities drive the most revenue, and how does that split by gender and age?

Build:

Matrix visual: Location (rows) → Gender (columns) → nested Age Group drill-down
Filters: `Subscription Status = Yes`, `Frequency Segment = High Frequency`
`Location` filtered to Top 10 by Total Revenue
`Total Revenue` added as a value, sorted descending, shown as % of grand total
Result: Narrowed from 2,581 total customers down to 240 high-value customers (119 Female, 121 Male) across the top 10 revenue-driving cities, generating ₹30,52,265 in revenue — with a full gender and age breakdown available on drill-down.


Top 10 cities by revenue, filtered to subscribed, high-frequency customers, broken down by gender with age-group drill-down and % of total revenue per city.
Key finding: Chennai leads this segment at 20.08% of total revenue (₹6,12,888 from 35 customers), followed by Hyderabad (13.48%) and Pune (12.02%) — together the top 3 cities account for ~45% of revenue from this high-value segment, despite representing less than a third of the 10 cities shown. Male customers slightly outspend female customers overall in this segment (₹15,48,067 vs ₹15,04,198), though the gap narrows or reverses in specific cities like Chennai and Hyderabad.
---

Next Steps (In Progress)
Discount impact analysis — total discount given, and whether discounting drives incremental orders or just erodes margin (`SUMX` over `Purchase Amount × Discount %`)
Return rate analysis — return % by category and by city, to flag operational risk areas
Additional dashboard pages: Category/Product performance, Seasonal trends, Delivery/logistics view
---**

Key Skills Demonstrated**
Data cleaning with intent (distinguishing meaningful nulls from data errors)
Star schema design from a single flat source file
Composite key construction for multi-column relationships
DAX: calculated columns vs. measures, `SWITCH(TRUE())`, `DIVIDE`, `DISTINCTCOUNT`, `SELECTEDVALUE` trade-offs
Top N filtering, drill-down hierarchies, % of grand total visualizations
