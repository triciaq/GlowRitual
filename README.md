# GlowRitual — Ecommerce Analytics Project

> **End-to-end customer analytics for a DTC beauty brand** — data cleaning, cohort retention analysis, and LTV modelling using real-world messy Shopify and Klaviyo exports.

---

## Project Overview

GlowRitual is a fictional Direct-to-Consumer (DTC) skincare brand used as a practice case study. The brand had a strong hero product (Vitamin C Brightening Serum) but suffered from high customer churn — 78% of buyers never returned for a second purchase.

This project simulates the full analytics workflow an ecommerce data analyst would run on a real client engagement:

- Receiving raw, messy exports from Shopify and Klaviyo
- Auditing and cleaning the data before any analysis
- Building a cohort retention table to quantify the churn problem
- Modelling Customer Lifetime Value (LTV) to size the revenue opportunity
- Producing insights ready to present to the client

---

## Dataset

Four raw files simulating a real client data dump:

| File | Source | Rows | Description |
|---|---|---|---|
| `GlowRitual_Orders_Raw.xlsx` | Shopify Admin → Orders → Export | 1,818 | One row per line item. Contains duplicates, mixed date formats, currency symbols in numeric fields, blank rows, accidental totals row |
| `GlowRitual_Customers_Raw.xlsx` | Shopify Admin → Customers → Export | 680 | One row per customer. Contains missing IDs, inconsistent capitalisation, mixed accepts-marketing formats |
| `GlowRitual_Products_Raw.xlsx` | Shopify Admin → Products → Export | 8 products + 96 inventory movements | Contains missing cost prices, inconsistent status values, HTML in description fields |
| `GlowRitual_Klaviyo_Raw.xlsx` | Klaviyo → Campaigns + Flows | 31 campaigns, 10 flows | Contains duplicate campaign sends, metrics stored as comma-strings, mixed date formats, mixed percentage formats |

### Data Quality Issues Found and Fixed

| Issue | Column(s) | Fix Applied |
|---|---|---|
| Currency symbols in numeric fields (`£45.00` stored as text) | Unit Price, Shipping, Order Total | `str.replace()` + `pd.to_numeric()` |
| Mixed date formats (5 different formats in one column) | Order Date | `pd.to_datetime(dayfirst=True)` |
| Duplicate rows (Shopify line-item export behaviour) | All | `drop_duplicates()` |
| Accidental totals row at bottom of export | Row 1819 | Identified via `Ctrl+End`, deleted |
| Blank rows from filtered paste | All | `dropna(subset=['Order ID'])` |
| Inconsistent product names (same SKU, 4–5 spellings) | Product Name | Mapped to SKU master |

---

## Tech Stack

```
Python 3.x
pandas
openpyxl
Jupyter Notebook
```

---

## Project Structure

```
glowritual-analytics/
│
├── data/
│   ├── raw/
│   │   ├── GlowRitual_Orders_Raw.xlsx
│   │   ├── GlowRitual_Customers_Raw.xlsx
│   │   ├── GlowRitual_Products_Raw.xlsx
│   │   └── GlowRitual_Klaviyo_Raw.xlsx
│   └── clean/
│       └── GlowRitual_Cohort_Analysis.xlsx
│
├── notebooks/
│   └── GlowRitual_Analysis.ipynb
│
└── README.md
```

---

## Analysis Walkthrough

### Step 1 — Data Audit
Before writing a single formula, audit every file:
- Count rows and columns
- Check data types per column (`df.dtypes`)
- Identify nulls (`df.isnull().sum()`)
- Spot structural issues (totals rows, blank rows, duplicates)

### Step 2 — Data Cleaning

```python
# Strip currency symbols and convert to numeric
for col in ['Unit Price', 'Shipping', 'Order Total']:
    orders[col] = orders[col].astype(str).str.replace('£','').str.replace('$','').str.replace(',','').str.strip()
    orders[col] = pd.to_numeric(orders[col], errors='coerce')

# Parse messy dates
orders['Order Date'] = pd.to_datetime(orders['Order Date'], dayfirst=True, errors='coerce')

# Remove duplicates and blank rows
orders = orders.drop_duplicates()
orders = orders.dropna(subset=['Order ID'])
orders = orders[~orders['Order ID'].astype(str).str.contains('TOTAL', na=False)]
```

**Result:** 1,818 raw rows → 1,743 clean rows (removed 69 duplicates, 6 blank rows, 1 totals row)

### Step 3 — Cohort Analysis

Tag every customer with their first purchase month (cohort), then track whether they returned in subsequent months.

```python
# Get each customer's cohort month
first_purchase = orders.groupby('Customer ID')['Order Date'].min().reset_index()
first_purchase['Cohort Month'] = first_purchase['First Purchase Date'].dt.to_period('M')

# Merge back onto orders
orders = orders.merge(first_purchase[['Customer ID','Cohort Month']], on='Customer ID', how='left')
orders['Order Month'] = orders['Order Date'].dt.to_period('M')

# Build cohort retention table
cohort_data = orders.groupby(['Cohort Month','Order Month'])['Customer ID'].nunique().reset_index()
cohort_data['Period'] = (cohort_data['Order Month'] - cohort_data['Cohort Month']).apply(lambda x: x.n)

cohort_pivot = cohort_data.pivot_table(index='Cohort Month', columns='Period', values='Customers')
retention_table = cohort_pivot.divide(cohort_pivot[0], axis=0).round(3) * 100
```

**Key finding:** Month 1 retention across all cohorts ranged from **6.2% to 12.9%** — confirming that the majority of customers churn within the first 30 days of acquisition.

### Step 4 — LTV Modelling

```python
orders['Line Revenue'] = orders['Qty'] * orders['Unit Price']

ltv_df = orders.groupby('Customer ID').agg(
    Total_Revenue = ('Line Revenue', 'sum'),
    Total_Orders  = ('Order ID', 'nunique'),
    First_Purchase= ('Order Date', 'min'),
    Last_Purchase = ('Order Date', 'max')
).reset_index()

ltv_df['Lifespan_Days'] = (ltv_df['Last_Purchase'] - ltv_df['First_Purchase']).dt.days
ltv_df['AOV'] = (ltv_df['Total_Revenue'] / ltv_df['Total_Orders']).round(2)
```

---

## Key Findings

| Metric | Value |
|---|---|
| Total unique customers | 680 |
| One-time buyers | 386 (56.8%) |
| Repeat buyers (2+ orders) | 294 (43.2%) |
| Average Order Value (AOV) | £70.50 |
| Average orders per customer | 1.90 |
| Peak acquisition month | January 2023 (103 new customers) |
| Worst Month-1 retention | August cohort: 6.2% |
| Best Month-1 retention | May cohort: 12.9% |

### Cohort Retention Highlights

```
Period          0       1       2       3       4       5
Cohort Month
2023-01      100.0     8.7    12.6    13.6    16.5    12.6
2023-02      100.0    10.4    16.4    14.9    20.9    28.4
2023-03      100.0     9.2     9.2    16.1    13.8     8.0
2023-06      100.0     8.0    12.5    15.9    17.0     4.5
```

Month 1 retention below 13% across all cohorts confirms the churn problem is concentrated in the first 30 days post-purchase — the window where a post-purchase email flow would have the highest impact.

---

## Business Implications

**The problem is not acquisition — it is what happens after the first purchase.**

GlowRitual was spending on ads to acquire customers who bought once and disappeared. The data shows:

1. **Month 1 is the critical window.** Customers who come back in month 1 show much stronger long-term retention in later periods. A well-built post-purchase email sequence targeting the first 30 days is the highest-leverage intervention.

2. **Repeat buyers are genuinely loyal.** Customers who made it past their first purchase went on to buy 3, 4, 5, 6 times. The product quality is not the problem.

3. **The email list was underperforming.** Klaviyo data shows the Win-Back flow was paused, the post-purchase flow was only partially active, and no replenishment reminder existed — despite the hero serum lasting approximately 45 days.

---

## What's Next (In Progress)

- [ ] Complete LTV model with right-censoring correction for late cohorts
- [ ] RFM segmentation (Champions, Loyal, At-Risk, Lost)
- [ ] Klaviyo email audit — open rates, click rates, revenue attribution by flow
- [ ] Product-level analysis — which SKU produces the most loyal buyers
- [ ] Client-ready summary report

---

## Author

**Tricia | Ecommerce Analytics Consultant**
Specialising in DTC beauty brands on Shopify ($200K–$2M revenue stage)

📧 patricianaggayi9@gmail.com
🔗 [linkedin.com/in/patricia-naggayi-277822204](https://linkedin.com/in/patricia-naggayi-277822204)

---

*This project is part of a portfolio of ecommerce analytics case studies. All brand data is fictional and generated for practice purposes.*
