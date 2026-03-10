# 🛍️ Customer Shopping Behavior Analysis

> An end-to-end data analytics project covering data cleaning, SQL business analysis, and interactive Power BI visualization — built as a portfolio project to demonstrate real-world analyst skills.

---

## 📌 Table of Contents

- [Project Overview](#project-overview)
- [Dataset](#dataset)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Phase 1 — Data Cleaning & EDA (Python)](#phase-1--data-cleaning--eda-python)
- [Phase 2 — SQL Business Analysis](#phase-2--sql-business-analysis)
- [Phase 3 — Power BI Dashboard](#phase-3--power-bi-dashboard)
- [Key Findings](#key-findings)
- [Business Recommendations](#business-recommendations)
- [How to Run](#how-to-run)

---

## Project Overview

This project analyzes customer shopping behavior across 3,900 transactions to uncover patterns in spending, product preferences, customer segmentation, and the impact of discounts and promotions.

The analysis follows a complete analytics pipeline:

```
Raw CSV Data → Python (Clean & EDA) → PostgreSQL (SQL Analysis) → Power BI (Dashboard)
```

**Goals:**
- Identify high-value customer segments
- Understand the effect of discounts on purchase behavior
- Analyze revenue distribution by gender, age group, and category
- Surface actionable business recommendations backed by data

---

## Dataset

| Property | Details |
|---|---|
| **Source** | Kaggle — Customer Shopping Trends Dataset |
| **Records** | 3,900 transactions |
| **Features** | 18 columns |
| **Domain** | Retail / E-commerce |

### Feature Overview

| Group | Columns |
|---|---|
| Customer Demographics | `age`, `gender`, `location`, `subscription_status` |
| Purchase Details | `item_purchased`, `category`, `purchase_amount`, `season`, `size`, `color` |
| Shopping Behavior | `discount_applied`, `previous_purchases`, `frequency_of_purchases`, `review_rating`, `shipping_type`, `payment_method` |

> **Note:** `promo_code_used` was dropped after analysis confirmed near-perfect correlation with `discount_applied` (redundant feature).

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **Python** (Pandas, NumPy) | Data cleaning, feature engineering, EDA |
| **PostgreSQL** | Business queries and data analysis |
| **SQLAlchemy** | Loading cleaned DataFrame into PostgreSQL |
| **Power BI Desktop** | Interactive dashboard and visualization |

---

## Project Structure

```
customer-shopping-analysis/
│
├── data/
│   ├── raw/
│   │   └── shopping_trends.csv          # Original dataset
│   └── cleaned/
│       └── shopping_trends_clean.csv    # Cleaned output
│
├── notebooks/
│   └── eda_cleaning.ipynb               # Python EDA notebook
│
├── sql/
│   └── business_analysis.sql            # All 10 SQL queries
│
├── dashboard/
│   └── customer_behavior.pbix           # Power BI file
│
└── README.md
```

---

## Phase 1 — Data Cleaning & EDA (Python)

### Steps Performed

**1. Initial Inspection**
```python
df.info()        # Check dtypes and null counts
df.describe()    # Statistical summary
df.isnull().sum() # Null value check
```

**2. Missing Value Treatment**

37 null values found in `review_rating` (~0.95% of rows). Imputed using **median per product category** to avoid global distribution skew.

```python
df['review_rating'] = df.groupby('category')['review_rating'] \
                        .transform(lambda x: x.fillna(x.median()))
```

**3. Column Standardization**

Renamed all columns to `snake_case` for SQL compatibility.

```python
df.columns = df.columns.str.lower().str.replace(' ', '_').str.replace('(', '').str.replace(')', '')
# e.g. "Purchase Amount (USD)" → "purchase_amount"
```

**4. Feature Engineering**

Created `age_group` by binning ages into meaningful segments:

```python
bins   = [0, 25, 40, 55, 100]
labels = ['Young Adult', 'Adult', 'Middle-aged', 'Senior']
df['age_group'] = pd.cut(df['age'], bins=bins, labels=labels)
```

**5. Redundancy Check**

`promo_code_used` and `discount_applied` were ~99% correlated — `promo_code_used` was dropped.

**6. Export to PostgreSQL**

```python
from sqlalchemy import create_engine
engine = create_engine("postgresql://user:password@localhost:5432/shopping_db")
df.to_sql('public_customer', engine, if_exists='replace', index=False)
```

---

## Phase 2 — SQL Business Analysis

All queries were run on PostgreSQL against the `public_customer` table.

---

### Q1 — Revenue by Gender

```sql
SELECT gender,
       ROUND(SUM(purchase_amount)::numeric, 2) AS total_revenue
FROM public_customer
GROUP BY gender
ORDER BY total_revenue DESC;
```

| Gender | Total Revenue (USD) |
|---|---|
| Male | $157,890 |
| Female | $75,191 |

> Male customers generated ~2.1x the revenue of female customers — reflecting a larger male customer proportion rather than higher per-transaction spend.

---

### Q2 — High-Spending Discount Users

```sql
SELECT COUNT(*) AS high_spenders_with_discount
FROM public_customer
WHERE discount_applied = 'Yes'
  AND purchase_amount > (SELECT AVG(purchase_amount) FROM public_customer);
```

| Result |
|---|
| **839 customers** applied discounts but still spent above average ($59.76) |

> These customers are prime targets for a loyalty programme — they're deal-conscious but genuinely high-value buyers.

---

### Q3 — Top 5 Products by Average Review Rating

```sql
SELECT item_purchased,
       ROUND(AVG(review_rating)::numeric, 2) AS avg_rating
FROM public_customer
GROUP BY item_purchased
ORDER BY avg_rating DESC
LIMIT 5;
```

| Rank | Product | Avg. Rating |
|---|---|---|
| 1 | Gloves | 3.86 |
| 2 | Sandals | 3.84 |
| 3 | Boots | 3.82 |
| 4 | Hat | 3.80 |
| 5 | Skirt | 3.78 |

---

### Q4 — Shipping Type vs. Average Purchase Amount

```sql
SELECT shipping_type,
       ROUND(AVG(purchase_amount)::numeric, 2) AS avg_purchase
FROM public_customer
GROUP BY shipping_type
ORDER BY avg_purchase DESC;
```

| Shipping Type | Avg. Purchase Amount |
|---|---|
| Express | $60.48 |
| Standard | $58.46 |

> Express shipping customers spend ~$2 more on average — a potential upsell opportunity at checkout.

---

### Q5 — Subscribers vs. Non-Subscribers

```sql
SELECT subscription_status,
       COUNT(*) AS total_customers,
       ROUND(AVG(purchase_amount)::numeric, 2) AS avg_spend,
       ROUND(SUM(purchase_amount)::numeric, 2) AS total_revenue
FROM public_customer
GROUP BY subscription_status;
```

| Status | Customers | Avg. Spend | Total Revenue |
|---|---|---|---|
| No | 2,847 | $59.87 | $170,436 |
| Yes | 1,053 | $59.49 | $62,645 |

> Average spend is nearly identical between groups — subscription does not appear to drive higher spend per transaction.

---

### Q6 — Discount-Dependent Products

```sql
SELECT item_purchased,
       ROUND(100.0 * SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) / COUNT(*), 2)
       AS discount_rate_pct
FROM public_customer
GROUP BY item_purchased
ORDER BY discount_rate_pct DESC
LIMIT 5;
```

| Rank | Product | Discount Rate |
|---|---|---|
| 1 | Hat | 50.00% |
| 2 | Sneakers | 49.66% |
| 3 | Coat | 49.07% |
| 4 | Sweater | 48.17% |
| 5 | Pants | 47.37% |

> ~50% of Hat and Sneaker purchases involve a discount — suggesting these products may be over-reliant on promotions to convert.

---

### Q7 — Customer Segmentation by Purchase History

```sql
SELECT
  CASE
    WHEN previous_purchases >= 10 THEN 'Loyal'
    WHEN previous_purchases BETWEEN 5 AND 9 THEN 'Returning'
    ELSE 'New'
  END AS customer_segment,
  COUNT(*) AS count
FROM public_customer
GROUP BY customer_segment
ORDER BY count DESC;
```

| Segment | Customers | Share |
|---|---|---|
| Loyal | 3,116 | 79.9% |
| Returning | 701 | 18.0% |
| New | 83 | 2.1% |

> Strong retention (79.9% Loyal), but very low new customer acquisition (2.1%) — growth opportunity through targeted campaigns.

---

### Q8 — Top 3 Products per Category (Window Function)

```sql
WITH ranked AS (
  SELECT category, item_purchased, COUNT(*) AS sales,
         RANK() OVER (PARTITION BY category ORDER BY COUNT(*) DESC) AS rnk
  FROM public_customer
  GROUP BY category, item_purchased
)
SELECT category, item_purchased, sales, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY category, rnk;
```

| Category | Rank 1 | Rank 2 | Rank 3 |
|---|---|---|---|
| Accessories | Jewelry (171) | Sunglasses (161) | Belt (161) |
| Clothing | Blouse (171) | Pants (171) | Shirt (169) |
| Footwear | Sandals (160) | Shoes (150) | Sneakers (145) |
| Outerwear | Jacket (163) | Coat (161) | Hoodie (152) |

---

### Q9 — Repeat Buyers & Subscription Correlation

```sql
SELECT subscription_status,
       COUNT(*) AS repeat_buyers
FROM public_customer
WHERE previous_purchases > 5
GROUP BY subscription_status
ORDER BY repeat_buyers DESC;
```

| Subscription | Repeat Buyers (>5 Purchases) |
|---|---|
| No | 2,518 |
| Yes | 958 |

> 72% of repeat buyers are non-subscribers — engaged, loyal customers who haven't converted to subscription. The subscription offer may need a stronger value proposition.

---

### Q10 — Revenue by Age Group

```sql
SELECT age_group,
       ROUND(SUM(purchase_amount)::numeric, 2) AS total_revenue
FROM public_customer
GROUP BY age_group
ORDER BY total_revenue DESC;
```

| Age Group | Total Revenue | Share |
|---|---|---|
| Young Adult | $62,143 | 26.7% |
| Middle-aged | $59,197 | 25.5% |
| Adult | $55,978 | 24.1% |
| Senior | $55,763 | 24.0% |

> Revenue is nearly equal across all age groups — the product range has broad demographic appeal.

---

## Phase 3 — Power BI Dashboard

The dashboard consolidates all findings into a single interactive view with the following components:

| Visual | Description |
|---|---|
| **KPI Cards** | Total Customers (3.9K), Avg. Purchase Amount ($59.76), Avg. Review Rating (3.75) |
| **Donut Chart** | Subscription split — Yes: 27%, No: 73% |
| **Bar Charts** | Revenue and Sales by Category (Clothing & Accessories lead) |
| **Horizontal Bars** | Revenue and Sales by Age Group |
| **Slicers** | Gender, Category, Subscription Status, Shipping Type for dynamic filtering |

**Dashboard Screenshot:**

![Customer Behavior Dashboard](dashboard/dashboard_screenshot.png)

> *Power BI Desktop file available in `/dashboard/customer_behavior.pbix`*

---

## Key Findings

- **Revenue is male-skewed** — Male customers account for ~68% of total revenue, likely reflecting a larger proportion in the dataset
- **Discounts drive volume but attract quality buyers** — 839 discount users still spend above average
- **Subscription has low conversion** — Only 27% of customers subscribe, despite near-identical spending behavior
- **New customer acquisition is weak** — Just 2.1% of customers are new, suggesting a growth gap
- **Revenue is evenly spread across age groups** — Each segment contributes ~24–27%, indicating broad product-market fit
- **50% of Hat and Sneaker sales rely on discounts** — Potential margin risk if discount strategy isn't reviewed

---

## Business Recommendations

| # | Recommendation | Rationale |
|---|---|---|
| 1 | **Launch a subscription conversion campaign** | 2,518 repeat non-subscribers are ideal targets — add exclusive perks like free express shipping |
| 2 | **Invest in new customer acquisition** | New customers = 2.1% of base; targeted ads for Young Adults could shift this meaningfully |
| 3 | **Review discount strategy for Hat & Sneakers** | ~50% discount dependency threatens margin; A/B test reduced discount rates |
| 4 | **Upsell express shipping at checkout** | Express users spend $2 more per order — a small prompt could increase basket size |
| 5 | **Feature top-rated products in campaigns** | Gloves, Sandals, Boots lead on review ratings — use as social proof in marketing |

---

## How to Run

### 1. Clone the Repository
```bash
git clone https://github.com/yourusername/customer-shopping-analysis.git
cd customer-shopping-analysis
```

### 2. Install Python Dependencies
```bash
pip install pandas numpy sqlalchemy psycopg2
```

### 3. Run the Cleaning Script
```bash
python notebooks/eda_cleaning.py
```

### 4. Set Up PostgreSQL
```bash
# Create database
createdb shopping_db

# The cleaning script auto-loads data via SQLAlchemy
# Update the connection string in the script if needed:
# postgresql://user:password@localhost:5432/shopping_db
```

### 5. Run SQL Queries
```bash
psql -d shopping_db -f sql/business_analysis.sql
```

### 6. Open Power BI Dashboard
Open `dashboard/customer_behavior.pbix` in **Power BI Desktop**.

---

## Author

**Palash**
Data Analyst | Banking Domain Background | Python · SQL · Power BI · Excel

> *This project was built as part of a structured self-learning roadmap toward a data analyst role.*

---

*Tools: Python · PostgreSQL · Power BI Desktop*
