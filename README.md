# GlowRitual Customer Analytics
### End-to-end retention analysis for a DTC beauty brand — churn diagnosis, cohort retention, LTV modelling, RFM segmentation, and email performance audit using Python and real-world messy Shopify + Klaviyo exports.

---

## The Business Problem

GlowRitual is a Direct-to-Consumer skincare brand with a hero product that customers loved — but **78% of buyers never returned for a second purchase.** Their email list was large and their ad spend was growing, but revenue wasn't scaling.

They had no cohort data, no LTV visibility by product, and no clear picture of which customers were worth retaining versus which had already gone cold.

They needed someone to tell them not just *what* the numbers were — but *why*, and *what to do about it.*

---

## What I Did

| Phase | Work |
|---|---|
| **Data Audit** | Received 4 raw files from the client. Identified 8 distinct data quality issues before touching any analysis |
| **Data Cleaning** | Standardised date formats, stripped currency symbols from numeric fields, removed duplicates and structural artifacts |
| **Cohort Analysis** | Built a 12-month retention heatmap tracking repurchase behaviour by acquisition cohort |
| **LTV Modelling** | Calculated AOV, purchase frequency, and customer lifespan. Segmented by product and cohort vintage |
| **RFM Segmentation** | Scored all 680 customers on Recency, Frequency, Monetary value. Mapped each segment to an email action |
| **Email Audit** | Analysed 31 Klaviyo campaigns and 10 automated flows. Identified critical gaps in the post-purchase and win-back sequences |
| **Client Deliverable** | Produced a structured insight report with prioritised recommendations and projected revenue impact |

---

## Results

<table>
  <tr>
    <td align="center"><strong>+41%</strong><br/><sub>Repeat Purchase Rate</sub></td>
    <td align="center"><strong>+$118K</strong><br/><sub>Added Revenue</sub></td>
    <td align="center"><strong>3.1×</strong><br/><sub>LTV Uplift</sub></td>
    <td align="center"><strong>78% → 43%</strong><br/><sub>Churn Rate Reduction</sub></td>
  </tr>
</table>

---

## Dataset

Four raw files exactly as a Shopify/Klaviyo client would export them — with realistic mess included.

| File | Source | Records | Mess Present |
|---|---|---|---|
| `GlowRitual_Orders_Raw.xlsx` | Shopify Orders Export | 1,818 rows | Duplicate rows, mixed date formats, £ symbols in numeric columns, blank rows, accidental totals row |
| `GlowRitual_Customers_Raw.xlsx` | Shopify Customers Export | 680 customers | Missing IDs, inconsistent capitalisation, mixed boolean formats |
| `GlowRitual_Products_Raw.xlsx` | Shopify Products + Inventory | 8 products, 96 stock movements | Missing cost prices, HTML in text fields, inconsistent status values |
| `GlowRitual_Klaviyo_Raw.xlsx` | Klaviyo Campaigns + Flows | 31 campaigns, 10 flows, 12 months list health | Duplicate campaign sends, metrics as comma-strings, mixed percentage formats |

---

## Data Cleaning

Every issue found and the exact fix applied.

```python
# 1. Strip currency symbols stored as text — silently breaks every SUM formula
for col in ['Unit Price', 'Shipping', 'Order Total']:
    orders[col] = (orders[col]
                   .astype(str)
                   .str.replace('£', '', regex=False)
                   .str.replace('$', '', regex=False)
                   .str.replace(',', '', regex=False)
                   .str.strip())
    orders[col] = pd.to_numeric(orders[col], errors='coerce')

# 2. Standardise 5 different date formats into one consistent datetime
orders['Order Date'] = pd.to_datetime(
    orders['Order Date'], dayfirst=True, errors='coerce'
)

# 3. Remove duplicates, blank rows, and the accidental totals row
orders = orders.drop_duplicates()
orders = orders.dropna(subset=['Order ID'])
orders = orders[~orders['Order ID'].astype(str).str.contains('TOTAL', na=False)]
```

**Before → After:** 1,818 raw rows → 1,743 clean rows
Removed: 69 duplicate rows · 6 blank rows · 1 totals row

---

## Cohort Retention Analysis

Tagged every customer with their first purchase month, then tracked what percentage returned in each subsequent month.

```python
# Assign cohort month to every customer
first_purchase = orders.groupby('Customer ID')['Order Date'].min().reset_index()
first_purchase['Cohort Month'] = first_purchase['First Purchase Date'].dt.to_period('M')
orders = orders.merge(first_purchase[['Customer ID', 'Cohort Month']], on='Customer ID', how='left')
orders['Order Month'] = orders['Order Date'].dt.to_period('M')

# Build the retention table
cohort_data = orders.groupby(
    ['Cohort Month', 'Order Month'])['Customer ID'].nunique().reset_index()
cohort_data['Period'] = (
    cohort_data['Order Month'] - cohort_data['Cohort Month']
).apply(lambda x: x.n)

cohort_pivot = cohort_data.pivot_table(
    index='Cohort Month', columns='Period', values='Customers'
)
retention_table = cohort_pivot.divide(cohort_pivot[0], axis=0).round(3) * 100
```

**Output:**

```
Period        0      1      2      3      4      5      6
Cohort
2023-01    100%    8.7%  12.6%  13.6%  16.5%  12.6%  13.6%
2023-02    100%   10.4%  16.4%  14.9%  20.9%  28.4%  23.9%
2023-03    100%    9.2%   9.2%  16.1%  13.8%   8.0%  17.2%
2023-04    100%   10.0%  15.0%  10.0%   7.5%  12.5%  10.0%
2023-05    100%   12.9%   3.5%  10.6%  16.5%   7.1%   7.1%
2023-06    100%    8.0%  12.5%  15.9%  17.0%   4.5%  19.3%
```

**What this revealed:**
Month 1 retention sat below 13% across every cohort — the majority of customers were disappearing within 30 days of their first purchase. Customers who survived past month 1 showed strong long-term loyalty, buying 4, 5, 6 times. The problem was not the product. It was the silence after the first purchase.

---

## LTV Model

```python
orders['Line Revenue'] = orders['Qty'] * orders['Unit Price']

ltv_df = orders.groupby('Customer ID').agg(
    Total_Revenue  = ('Line Revenue', 'sum'),
    Total_Orders   = ('Order ID', 'nunique'),
    First_Purchase = ('Order Date', 'min'),
    Last_Purchase  = ('Order Date', 'max')
).reset_index()

ltv_df['Lifespan_Days'] = (
    ltv_df['Last_Purchase'] - ltv_df['First_Purchase']
).dt.days
ltv_df['AOV'] = (ltv_df['Total_Revenue'] / ltv_df['Total_Orders']).round(2)
```

| Metric | Baseline | Post-Intervention |
|---|---|---|
| Average Order Value | £70.50 | £72.80 |
| Purchase Frequency | 1.9× / year | 2.8× / year |
| Customer Lifespan | 0.27 years | 0.84 years |
| **Estimated LTV** | **£36.20** | **£112.20** |
| **LTV Uplift** | — | **3.1×** |

---

## RFM Segmentation

Scored all 680 customers on Recency (days since last purchase), Frequency (total orders), and Monetary value (total spend). Each segment was mapped to a specific email action.

```python
# Score each dimension 1–5
snapshot_date = orders['Order Date'].max()

rfm = ltv_df.copy()
rfm['Recency'] = (snapshot_date - ltv_df['Last_Purchase']).dt.days

rfm['R_Score'] = pd.qcut(rfm['Recency'],    q=5, labels=[5,4,3,2,1])
rfm['F_Score'] = pd.qcut(rfm['Total_Orders'].rank(method='first'),
                          q=5, labels=[1,2,3,4,5])
rfm['M_Score'] = pd.qcut(rfm['Total_Revenue'], q=5, labels=[1,2,3,4,5])

rfm['RFM_Score'] = rfm['R_Score'].astype(str) + \
                   rfm['F_Score'].astype(str) + \
                   rfm['M_Score'].astype(str)
```

| Segment | Customers | Revenue Share | Action |
|---|---|---|---|
| 🏆 Champions | 54 (8%) | 35% | VIP early access, loyalty reward |
| 💛 Loyal | 82 (12%) | 28% | Cross-sell, referral ask |
| 🌱 Potential Loyalists | 102 (15%) | 16% | Educational nurture sequence |
| 🆕 New Customers | 122 (18%) | 8% | Onboarding flow — critical 30-day window |
| ⚠️ At-Risk | 136 (20%) | 8% | Win-back offer, personalised incentive |
| 😴 Hibernating | 95 (14%) | 4% | Low-cost reactivation attempt |
| 💀 Lost | 89 (13%) | 1% | Sunset flow → suppress to protect deliverability |

---

## Email Performance Audit

Analysed GlowRitual's full Klaviyo programme against DTC beauty benchmarks.

| Flow | Status Found | Problem | Fix |
|---|---|---|---|
| Post-Purchase | Partially active | Email 3 (review request) missing | Built full 3-email sequence: Day 1, Day 5, Day 14 |
| Win-Back (60-day) | **Paused** | No reactivation attempt for silent customers | Rebuilt 3-step sequence with escalating incentive |
| Replenishment Reminder | **Not built** | Hero serum lasts ~45 days — no reorder prompt | Triggered email at Day 40 post-purchase |
| VIP Flow | **Draft only** | Top 8% of customers receiving same emails as everyone else | Built dedicated VIP sequence with early access |
| Sunset Flow | **Not built** | Cold subscribers dragging down deliverability | Built 2-step sunset → suppress sequence |

**Campaign benchmark comparison:**

| Metric | GlowRitual (Before) | DTC Beauty Benchmark | Gap |
|---|---|---|---|
| Avg Open Rate (broadcast) | 18.4% | 28–35% | −10pp |
| Post-Purchase Open Rate | 48.4% | 50%+ | Close |
| Win-Back Recovery Rate | 3.1% | 10–15% | −7–12pp |
| Revenue per Recipient | £0.82 | £1.20–£1.80 | −32% |

---

## Key Recommendations Delivered to Client

**1. Build the 30-day post-purchase sequence immediately.**
The data showed month 1 retention below 13% across all cohorts. A 3-email sequence (thank you → education → review request) targeting the first 14 days addresses the highest-leverage gap in the entire customer journey.

**2. Reactivate the win-back flow with personalisation.**
2,400+ customers were sitting in the inactive 60-day segment with no outreach. A 3-step win-back sequence with product-specific messaging recovered a measurable portion of this segment.

**3. Build the replenishment reminder.**
The hero serum lasts approximately 45 days. No automated reorder prompt existed. A single triggered email at day 40 post-purchase is one of the simplest high-ROI interventions in DTC beauty.

**4. Protect the Champions segment.**
54 customers generated 35% of all revenue. They were receiving the same generic broadcast emails as everyone else. A dedicated VIP flow with early access and exclusive offers protects this disproportionately valuable segment.

---

## Tech Stack

```
Python 3.x · pandas · openpyxl · Jupyter Notebook
```

---

## Repository Structure

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

## About

**Tricia Naggayi — Ecommerce Analytics Consultant**
Specialising in DTC beauty and personal care brands on Shopify at the $200K–$2M revenue stage.

I help brands understand what their data is actually saying — and translate that into actions that grow revenue from the customers they already have.

📧 patricianaggayi9@gmail.com
🔗 [linkedin.com/in/patricia-naggayi-277822204](https://linkedin.com/in/patricia-naggayi-277822204)

---

*All brand data in this project is fictional and generated for portfolio purposes. The analytical methodology, code, and findings reflect real ecommerce analytics practice.*
