# Decoding Customer Value: A SQL-Driven Retention Strategy

Summer Projects '26 | Consulting & Analytics Club, IIT Guwahati

A customer-intelligence project for a D2C fashion brand (about 3,900 customers, US-wide). It answers one question:

> Is the business building a loyal customer base, or is it reliant on continuous promotional activity, and what should it do in either case?

**Team:** _Ayush umare and antariksh dongre (23110346, 23110031)_

---

## Answer in brief

- The brand is **not** yet building loyalty through promotions. 43% of customers (1,677) carry a discount and promo code, but discounted men and full-price men look the same on prior purchases, order value, purchase frequency and rating (all p > 0.05).
- Every discounted customer is male. No woman (0 of 1,248) received a discount, and all 1,053 subscribers did, so subscription works as the discount vehicle.
- A genuine loyal core exists (28.3% of customers) and it is a full-price core.
- Recommendation: a two-wave promo sunset for 477 habitual discounted customers, and an ideal-customer profile built on behavior (full price, more than 25 prior purchases, about $80 per order).

Full write-up: [`reports/Retention_Playbook_and_Executive_Summary.docx`](reports/Retention_Playbook_and_Executive_Summary.docx)

---

## Repository structure

```
.
├── README.md
├── data/
│   ├── raw/Dataset.csv                     # original data, 3,900 rows x 18 columns
│   └── processed/enriched_dataset.csv      # cleaned + 13 engineered features (31 columns)
├── notebooks/
│   └── Decoding_Customer_Value_23110346_23110031.ipynb   # cleaning + feature engineering
├── sql/
│   ├── sql_query_for_all_3q.sql            # Q1-Q3 segmentation queries
│   ├── sql_query_q4_q5.sql                 # Q4-Q5: promo sunset sizing, ideal customer
│   └── outputs/                            # sql_query_q1.csv, q2.csv, q3.csv
├── dashboard/
│   └── Customer_Value_Retention_Founder_Dashboard.pbix   # Power BI, 4 panels
├── reports/
│   └── Retention_Playbook_and_Executive_Summary.docx     # 1-page summary + playbook
└── docs/
    └── Problem_Statements.pdf              # original brief
```

---

## What was done

### 1. Data preparation and feature engineering (Python)

Notebook: `notebooks/Decoding_Customer_Value_23110346_23110031.ipynb`

The dataset has no loyalty score, no churn label and no timestamps, so every concept is constructed from the available columns.

Cleaning: numeric gaps are filled with the column median, and every column was checked for its unique values. `Discount Applied` and `Promo Code Used` are identical in all rows, so they are treated as a single signal.

| Feature | Definition | Question it answers |
|---|---|---|
| `promo_dependency_score` | Mean of discount and promo-code flags (0 or 1 in practice) | Does this customer buy only with a discount? |
| `total_spend` | (Previous Purchases + 1) x Purchase Amount | How much revenue has this customer likely generated? (proxy) |
| `satisfaction_flag` | Review Rating >= 4 | Is the customer satisfied? |
| `purchase_frequency_score` | Frequency label mapped to purchases per year (Weekly = 52 ... Annually = 1) | How often do they buy? |
| `engagement_score` | Equal-weight mean of min-max frequency, min-max prior purchases, subscription flag | How committed are they to the brand? |
| `value_score`, `value_tier` | PCA on spend, engagement, promo dependency, satisfaction; split into thirds | Who deserves the most retention investment? |
| `loyalty_def_A` | Previous Purchases above the median (25) **and** no promo | Repeat buyer who does not need a discount |
| `loyalty_def_B` | Subscribed **and** buys at least monthly | Formally engaged member |
| `customer_segment` | Rule-based: Champion, Loyalist, Deal Hunter, At Risk, Potential Loyalist, Passive | Which action applies to whom |

### 2. Loyalty: two competing definitions, one chosen

The brief requires testing at least two definitions and arguing for one.

| Test | Definition A (repeat + no discount) | Definition B (subscribed + monthly+) |
|---|---|---|
| Customers flagged | 1,103 (28.3%) | 599 (15.4%) |
| Share discounted | 0% | 100% |
| Correlation with `total_spend` | 0.43 (partly circular, since it uses prior purchases) | 0.01 |
| Prior purchases, flagged vs not | 37.7 vs 20.5 | 26.4 vs 25.2 |
| Overlap | 0 customers (kappa = -0.25) | |

**Chosen: Definition A.** B labels discount recipients as loyal, because every subscriber is discounted, and it shows no link to spend. A is the only definition that separates repeat buying from discount reliance. Caveat: A inherits the gender pattern (all women are discount-free), and it does not predict current order value or rating on its own.

### 3. SQL segmentation layer

| File | Question |
|---|---|
| `sql_query_for_all_3q.sql` (Q1) | What separates high-value from low-value customers? |
| Q2 | Which season and category combinations go with high prior purchases? |
| Q3 | Which states show organic demand vs discount-driven volume? |
| `sql_query_q4_q5.sql` (Q4) | Who to stop discounting, and what is at stake? |
| Q5 | What does the ideal customer look like, and where are they concentrated? |

Queries are MySQL-style (`use project_db;`). Load `enriched_dataset.csv` into a table called `enriched_dataset`.

### 4. Founder dashboard (Power BI)

`dashboard/Customer_Value_Retention_Founder_Dashboard.pbix` has the four panels the brief asks for: customer pyramid, promo dependency vs retention by segment, geographic opportunity map, and category funnel.

_Add a screenshot here: `![Dashboard](dashboard/dashboard.png)`_

### 5. Retention playbook

- **Promo sunset plan.** Wave 1: 155 habitual discounted non-subscribers (Previous Purchases > 25, orders at least monthly), promo codes stopped. Wave 2: 322 habitual discounted subscribers, moved from a percentage discount to non-price perks. A protected group (317 discounted customers with 10 or fewer prior purchases) is left unchanged. Each wave has a trigger, timeline, holdout group and metric.
- **Margin impact.** At an assumed 15% discount depth and 55% gross margin, roughly $47K (Wave 1) and $95K (Wave 2) a year is recovered if volume holds. The plan breaks even at about 73% volume retention.
- **Ideal customer.** Definition A plus top-quartile lifetime spend: about 550 customers (14%), generating 28.7% of the spend proxy, with about $80 average order vs $57 for the rest. Category, season, payment method, shipping and age show no significant skew.

---

## How to reproduce

```bash
pip install pandas scikit-learn jupyter
# the notebook reads Dataset.csv from its working directory:
cd data/raw && jupyter notebook ../../notebooks/Decoding_Customer_Value_23110346_23110031.ipynb
```

Then load `data/processed/enriched_dataset.csv` into MySQL as `enriched_dataset` and run the files in `sql/`. Open the `.pbix` in Power BI Desktop.

---

## Limitations

- **No timestamps or discount depth.** "Discounted" means discounted on the latest order. Causal claims need the holdout test in the playbook.
- **`total_spend` is a proxy.** It assumes past orders cost the same as the current one.
- **Margin figures are assumptions.** The 10-20% discount depth and 55% gross margin are not in the data.
- **`value_tier` caveat.** The PCA gives promo dependency a positive weight, so the "High Value" tier is 90% discount-dependent. The playbook relies on the loyalty definitions instead, not on this tier.
- **Small state samples.** Each state has 63-96 customers, so state findings are test-market leads rather than conclusions.
- **Gender pattern.** Discounts are assigned only to men in this data, so gender findings reflect how discounts were assigned, not customer preference.

---

## Tools

Python (pandas, scikit-learn), MySQL, Power BI, Word.
