# Customer Segmentation & Revenue Forecasting

Customer segmentation and revenue forecasting built on 50,000 transaction records spanning 2 years (2024–2025).

## Contents

| File | Description |
|---|---|
| `master_sales.csv` | Transaction-level dataset (transactions, products, customers, campaigns) |
| `Module2_Customer_Analytics.ipynb` | RFM analysis, K-Means clustering, CLV estimation, repeat purchase & revenue concentration |
| `Module5_Revenue_Forecasting.ipynb` | Monthly trend/seasonality, holdout-validated forecasting model, residual diagnostics, 6-month forecast |

## Data

`master_sales.csv` — one row per transaction:
- **Transaction:** `transaction_id`, `transaction_date`, `quantity`, `unit_price`, `discount_percentage`, `gross_revenue`, `net_revenue`, `cost`, `gross_margin`, `payment_status`
- **Product:** `product_id`, `product_name`, `category`, `standard_price`, `cost_price`, `margin_percentage`, `active_status`
- **Customer:** `customer_id`, `customer_type`, `segment` (business-assigned: Premium/Standard/Budget/New), `acquisition_channel`, `customer_since`, `location_category`
- **Campaign:** `campaign_id`, `campaign_name`, `has_campaign`, `campaign_start_date`/`end_date`, `offer_type`, `campaign_cost`
- **Derived:** `transaction_year`, `transaction_month`, `transaction_year_month`, `discount_band`, `customer_tenure_days`

---

## Customer Analytics

### 1. RFM Analysis
The dataset already carries a business-assigned `segment` label (Premium / Standard / Budget / New) — but a label isn't an analysis. RFM measures actual behaviour independently so it can be checked against that label:

- **Recency** — days since each customer's last purchase, relative to a snapshot date one day after the final transaction.
- **Frequency** — number of transactions per customer.
- **Monetary** — total net revenue per customer.

Each metric is split into quintiles (`pd.qcut`) and scored 1–5 (Recency scored in reverse, since fewer days is better), since the three raw metrics live on incompatible scales (days, counts, currency). Scores are summed into `rfm_total` (range 3–15) and mapped to business-friendly value tiers.

**Finding:** a sizeable number of customers labelled `Budget`/`Standard` by the business actually score `High Value` on RFM, and vice versa for some `Premium` customers — the existing business label is not a reliable proxy for actual purchase value.

### 2. K-Means Clustering
The quintile tiers above use fixed hand-picked thresholds. Clustering instead lets the data define its own groupings, using the *raw* R/F/M values (not the 1–5 scores), standardized and tested across k = 2–8 via inertia (elbow) and silhouette score.

**Finding:** silhouette peaks at k=3, statistically validating the 3-tier RFM segmentation. **k=4 is selected for the final model anyway** — it retains an acceptable silhouette (~0.33, above the 0.25 rule-of-thumb) while separating out a distinct low-frequency, high-recency "at-risk" group that's commercially useful to flag on its own. This trade-off (granularity vs. pure statistical optimum) is stated explicitly rather than silently defaulting to the best-scoring k.

### 3. Customer Lifetime Value (CLV)
```
CLV = Average Order Value × Purchase Frequency per Year × Assumed Lifespan (years)
```
Lifespan is set to **3 years** as an explicit assumption — the dataset only covers 2 years of actual transactions, so this is a planning horizon, not an observed figure.

### 4. Repeat Purchase Behaviour & Inactive Customers
- Every customer already has ≥6 transactions, so a binary "bought more than once" flag adds no information here.
- **Inactive** is defined as Recency > 90 days — chosen relative to the data's own median purchase gap (16 days), making 90 days a meaningful drop-off rather than an arbitrary cutoff.

### 5. Top Customers & Revenue Concentration
Customers ranked by total net revenue to check what share the top decile contributes — a Pareto-style concentration check applied at the customer level.

Finding: revenue is spread fairly evenly across the customer base — the top 10% contribute well under half of total revenue, unlike a classic 80/20 pattern. The business is not overly dependent on a small handful of customers.

**Finding:** revenue is spread fairly evenly across the customer base — the top 10% contribute well under half of total revenue, unlike a classic 80/20 pattern. The business is not overly dependent on a small handful of customers, a different risk profile from the product-level concentration seen elsewhere.

---

## Revenue Forecasting

### 1. Monthly Trend & Seasonality Check
Monthly revenue is aggregated and plotted before any model is chosen. Revenue rises every November/December in both years and drops back in January — a clear annual seasonal pattern on top of a mild upward trend.

With only 24 months of history (exactly 2 seasonal cycles), models like Holt-Winters or SARIMA would have just enough data to initialise a seasonal component but no spare data to validate it — noted explicitly as a limitation. A **classical trend + seasonal-index decomposition** is used instead: it captures the same trend/seasonality without needing more data than is available, and is easier to explain to a business audience than a fitted ARIMA model.

### 2. Train/Test Split & Naive Baseline
The last 3 months are held out and never touched during fitting, giving an honest out-of-sample accuracy check. The **naive baseline** (average of the last 3 training months, carried forward flat) sets the bar any real model has to beat.

### 3. Trend + Seasonal-Index Model
A linear trend combined with monthly seasonal indices, fit on the training period only.

### 4. Forecast Accuracy on the Holdout Period
**Finding:** the trend + seasonal-index model achieves a much lower MAPE than the naive baseline on unseen data — confirming that explicitly modelling the Nov/Dec seasonal spike is worth the extra step over assuming next month looks like the recent average.

### 5. Residual Diagnostics
Residuals from the fitted model are checked via ACF (autocorrelation) and a Shapiro-Wilk normality test to confirm the model isn't leaving structure unexplained.

### 6. Final 6-Month Forecast with Scenario Bands
The trend line is refit on **all 24 months** (the holdout split was only needed to validate the method) and projected 6 months ahead. Best/base/worst-case bands are set using the model's own in-sample residual standard deviation — tying the scenario range to demonstrated model accuracy rather than an arbitrary ± percentage.

---

## Requirements
```
pandas
numpy
matplotlib
scikit-learn
scipy
statsmodels
```

## Usage
```bash
jupyter notebook Module2_Customer_Analytics.ipynb
jupyter notebook Module5_Revenue_Forecasting.ipynb
```
Keep all three files in the same directory — both notebooks read `master_sales.csv` directly.
