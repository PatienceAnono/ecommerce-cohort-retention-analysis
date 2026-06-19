# Shopify Cohort Retention Analysis

SQL-driven cohort retention analysis on e-commerce transaction data, with Python heatmap visualisations and a full business interpretation. The kind of analysis you would run when a client says "we're losing customers but we don't know when" — this shows exactly when, by how much, and what the revenue impact looks like.

---

## What this project does

Takes a flat orders table (structured like a Shopify CSV export), runs a multi-step SQL pipeline to build a cohort table, then visualises the retention data as heatmaps, trend lines and a cumulative revenue chart. The data generation script creates realistic synthetic Shopify-format data so the notebook runs out of the box — but you can drop in a real Shopify export and it works without changing a single line.

The SQL is the centrepiece. Everything from cohort assignment to retention rate calculation is done in SQL before any Python charting happens. That is intentional — in a real business setting, this query would live in BigQuery or your data warehouse and update automatically.

---

## Results from this run

**Dataset:** 6,404 orders · 2,810 customers · $401,672 total revenue · Oct 2023 – Dec 2024

**Retention rates (average across all cohorts):**

| Month | Retention Rate | What it means |
|-------|---------------|---------------|
| M0 | 100% | Acquisition month — everyone's in |
| M1 | 37.9% | Only 4 in 10 customers come back after the first month |
| M2 | ~24% | The steepest drop happens in the first two months |
| M3 | 18.7% | Starting to level out |
| M6 | 11.2% | Customers who reach here tend to stay |
| M12+ | ~9% | Long-term retention floor |

The M0 → M1 drop is the most important number in the whole analysis. Losing 62% of customers between the first and second month is the norm in e-commerce, but even a 5 percentage point improvement there — from 38% to 43% — would compound significantly across all cohorts.

**Top acquisition channel:** Google ($85,191 revenue)  
**Top product category:** Clothing & Apparel ($91,048 revenue)

---

## The SQL pipeline — how it works

The core logic is five sequential views built on the raw orders table. This is standard SQL that runs in SQLite, BigQuery, PostgreSQL or MySQL without modification.

**Step 1 — Assign every customer to a cohort**

```sql
CREATE VIEW v_first_orders AS
SELECT
    customer_id,
    MIN(order_date)              AS first_order_date,
    SUBSTR(MIN(order_date), 1, 7) AS cohort_month
FROM orders
GROUP BY customer_id;
```

Every customer belongs to exactly one cohort: the calendar month they first purchased. A customer who buys in October 2023 is in the Oct-2023 cohort forever, even if they keep buying through 2024.

**Step 2 — Calculate cohort index (months since first purchase)**

```sql
CREATE VIEW v_cohort_activity AS
SELECT
    o.customer_id,
    f.cohort_month,
    SUBSTR(o.order_date, 1, 7) AS order_month,
    (
        (CAST(SUBSTR(o.order_date, 1, 4) AS INTEGER) -
         CAST(SUBSTR(f.first_order_date, 1, 4) AS INTEGER)) * 12
        +
        (CAST(SUBSTR(o.order_date, 6, 2) AS INTEGER) -
         CAST(SUBSTR(f.first_order_date, 6, 2) AS INTEGER))
    ) AS cohort_index
FROM orders o
JOIN v_first_orders f ON o.customer_id = f.customer_id;
```

Month 0 = acquisition month. Month 1 = the calendar month after first purchase. Month 6 = six months later. This index lets us compare cohorts directly, regardless of when they were acquired.

**Step 3 — Count distinct active customers per cohort and month**

```sql
CREATE VIEW v_cohort_size AS
SELECT
    cohort_month,
    cohort_index,
    COUNT(DISTINCT customer_id) AS active_customers
FROM v_cohort_activity
GROUP BY cohort_month, cohort_index;
```

**Step 4 — Get original cohort sizes for the denominator**

```sql
CREATE VIEW v_cohort_totals AS
SELECT
    cohort_month,
    COUNT(DISTINCT customer_id) AS cohort_size
FROM v_first_orders
GROUP BY cohort_month;
```

**Step 5 — Calculate retention rates**

```sql
SELECT
    s.cohort_month,
    t.cohort_size,
    s.cohort_index,
    s.active_customers,
    ROUND(
        CAST(s.active_customers AS FLOAT) / CAST(t.cohort_size AS FLOAT) * 100, 1
    ) AS retention_rate_pct
FROM v_cohort_size s
JOIN v_cohort_totals t ON s.cohort_month = t.cohort_month
ORDER BY s.cohort_month, s.cohort_index;
```

The output of Step 5 is all you need. Pivot it in Python and you have the heatmap data. Every number in the heatmap comes directly from this query — no manipulation in Python beyond reshaping for the chart.

---

## Visuals

| File | What it shows |
|------|--------------|
| `1_cohort_retention_heatmap.png` | The main deliverable — retention % for every cohort × month combination. Darker blue = lower retention. Annotated with cohort sizes (n=X). M0 column highlighted in orange. |
| `2_retention_curves.png` | Left: all individual cohort curves overlaid. Right: average curve with ±1 standard deviation band. Shows where retention stabilises. |
| `3_revenue_heatmap.png` | Revenue per active customer by cohort and month. Shows whether retained customers spend more or less over time. |
| `4_channel_category.png` | Acquisition channel revenue comparison and product category performance with average order values. |
| `5_cohort_size_retention_trend.png` | Top: how many customers acquired each month. Bottom: M1 and M3 retention rate over time — are things getting better or worse? |
| `6_cumulative_revenue.png` | How total revenue accumulates for each cohort over time. Shows which acquisition months generate the most long-term value. |
| `7_executive_dashboard.png` | Single-page summary combining all key metrics for a leadership briefing. |

---

## Project structure

```
shopify-cohort-retention/
│
├── generate_data.py                      ← Creates realistic synthetic Shopify data
│
├── cohort_analysis.ipynb                 ← Main analysis notebook
│
├── sql/
│   └── cohort_retention_queries.sql      ← All SQL queries with comments
│                                            (portable to BigQuery, PostgreSQL, MySQL)
│
├── data/
│   ├── raw/
│   │   └── shopify_orders.csv            ← Shopify-format orders data
│   └── processed/
│       ├── cohort_retention_rates.csv    ← Full retention table (long format)
│       ├── cohort_pivot_retention.csv    ← Pivot table — heatmap ready
│       ├── cohort_revenue_summary.csv    ← Revenue metrics per cohort
│       └── cohort_retention_powerbi.csv  ← Power BI ready with labels
│
├── visuals/
│   ├── 1_cohort_retention_heatmap.png
│   ├── 2_retention_curves.png
│   ├── 3_revenue_heatmap.png
│   ├── 4_channel_category.png
│   ├── 5_cohort_size_retention_trend.png
│   ├── 6_cumulative_revenue.png
│   └── 7_executive_dashboard.png
│
└── README.md
```

---

## How to use with real Shopify data

This is the part that makes this project actually useful beyond a portfolio piece.

**Export from Shopify:**
1. Shopify Admin → Analytics → Reports → Orders
2. Export → All time → CSV
3. Drop the file into `data/raw/shopify_orders.csv`

**Column mapping** — if Shopify's export uses slightly different column names, update these two lines in the notebook:

```python
# Current expected columns
df['customer_id'] = df['customer_id']       # or 'Customer ID'
df['order_date']  = df['created_at']         # or 'Created at'
df['total_price'] = df['total_price']        # or 'Total'
```

Everything else runs without changes. The SQL views, the pivot logic, and all the charts will work on real data as long as those three columns are present and correctly named.

**For BigQuery or PostgreSQL:**
Copy the queries from `sql/cohort_retention_queries.sql` directly into your warehouse. Replace `SUBSTR()` with `FORMAT_DATE()` or `DATE_TRUNC()` as appropriate for your SQL dialect.

---

## How to run

```bash
git clone https://github.com/anonopatience/shopify-cohort-retention.git
cd shopify-cohort-retention

pip install pandas numpy matplotlib seaborn

# Generate the synthetic dataset
python generate_data.py

# Open the notebook
jupyter notebook cohort_analysis.ipynb
```

Or run the analysis directly as a script if you don't need Jupyter:

```bash
python cohort_analysis.ipynb  # if saved as .py
```

---

## What the heatmap is actually telling you

A few things worth understanding when reading the output:

**The diagonal matters.** Empty cells in the bottom-right of the heatmap are not missing data — they are periods that haven't happened yet. A cohort acquired in November 2024 cannot have a M6 retention figure in December 2024. The heatmap fills in over time.

**Early cohorts are more complete.** The October 2023 cohort has 14 months of data. The December 2024 cohort has 1. When comparing cohorts, only compare them on the months where both have data.

**M1 retention is the lever.** Almost every cohort has a similar shape: sharp drop from M0 to M1, smaller drops from M1 to M3, then gradual stabilisation. If you want to move the aggregate retention number, M1 is where to focus. The customers who survive M1 tend to stick around.

**Revenue per customer often goes up over time.** Retained customers frequently spend more in later months than they did at acquisition. That is why the revenue heatmap (chart 3) can look healthier than the retention heatmap even for later months — fewer customers, but higher-value ones.

---

## Business recommendations from this analysis

**1. Fix the M0 → M1 problem first.**
The biggest drop is always going to be between acquisition and the first repeat purchase. Improving this specific transition — through onboarding emails, post-purchase follow-up, or a first-repeat incentive — has the highest leverage of anything in this analysis.

**2. Identify what the loyal 11% have in common.**
Customers who reach M6 have a retention rate of around 11%. That is not a lot, but those customers are worth far more than their acquisition cost implies. Understand who they are — which channel brought them, which product category they bought first, which city tier they're in — and optimise acquisition toward that profile.

**3. Use the channel analysis strategically.**
Google delivers the highest revenue, but that includes both acquisition cost and lifetime value. The more useful question is: which channel brings customers who come back? That requires connecting channel data to cohort data at the individual customer level — the `source_name` column in the dataset allows this.

**4. Product category as a retention predictor.**
Some product categories naturally drive repeat purchases (consumables, skincare, food) and some don't (electronics, one-time purchases). If Clothing & Apparel is your top revenue category, the question is whether those customers are buying one outfit or coming back every season. The category × cohort breakdown in the SQL bonus queries helps answer this.

---

## Tools used

- **SQL (SQLite)** — full cohort pipeline from raw orders to retention rates
- **Python / Pandas** — data loading, pivot table construction, export
- **Seaborn / Matplotlib** — heatmaps, line charts, bar charts, dashboard
- **NumPy** — array operations for colour mapping and statistics

---

## About

**Patience Anono** — Data Analyst & Analytics Consultant

I work with e-commerce brands on analytics projects that go from raw transaction data to the business decisions that actually change outcomes.

📧 anonopatience@gmail.com  
🌐 [padataanalytics.com](https://padataanalytics.com)  
💼 [LinkedIn](https://www.linkedin.com/in/patience-anono-22ab06176/)  
💻 [GitHub]([https://github.com/anonopatience](https://github.com/PatienceAnono?tab=repositories)

---


