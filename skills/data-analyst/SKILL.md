---
name: data-analyst
description: Data analysis workflows: pandas/SQL patterns, statistical analysis, visualization (matplotlib, plotly), dashboard design, A/B test analysis, and business metrics.
---

# Data Analyst

Practical data analysis patterns for turning raw data into actionable insights. Covers the full analysis stack: data wrangling with pandas, analytical SQL, statistical testing, visualization, A/B experiment analysis, business metrics, and dashboard design. All examples use Python 3.11+ with modern library APIs.

## Table of Contents

- [Pandas Patterns](#pandas-patterns)
- [SQL for Analysis](#sql-for-analysis)
- [Statistical Analysis](#statistical-analysis)
- [Data Visualization](#data-visualization)
- [A/B Test Analysis](#ab-test-analysis)
- [Business Metrics](#business-metrics)
- [Dashboard Design](#dashboard-design)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Pandas Patterns

### Loading Data

```python
import pandas as pd

# CSV with explicit dtypes (avoid silent type inference)
df = pd.read_csv("events.csv", dtype={"user_id": "str", "event_type": "category"},
                  parse_dates=["created_at"], usecols=["user_id", "event_type", "amount", "created_at"])

# Parquet (preferred -- typed, compressed, fast)
df = pd.read_parquet("events.parquet", columns=["user_id", "amount"])

# SQL directly into DataFrame
from sqlalchemy import create_engine
engine = create_engine("postgresql://user:pass@localhost/analytics")
df = pd.read_sql("SELECT user_id, amount FROM events WHERE created_at >= '2025-01-01'",
                  engine, parse_dates=["created_at"])
```

### Cleaning: Missing Values and Dtypes

```python
# Inspect missing data
print(df.isnull().sum())
print(df.dtypes)

# Drop rows where key columns are null
df = df.dropna(subset=["user_id", "amount"])

# Fill missing values with context-appropriate defaults
df["region"] = df["region"].fillna("Unknown")
df["amount"] = df["amount"].fillna(0.0)

# Convert dtypes for memory efficiency and correctness
df["user_id"] = df["user_id"].astype("str")
df["region"] = df["region"].astype("category")  # ~90% less memory for low-cardinality
df["created_at"] = pd.to_datetime(df["created_at"], utc=True)

# Detect and remove duplicates
dupes = df.duplicated(subset=["user_id", "created_at"], keep="first")
print(f"Found {dupes.sum()} duplicates")
df = df[~dupes]
```

### Merging and Joining

```python
# Left join: keep all rows from the left DataFrame
merged = pd.merge(
    events,
    users[["user_id", "plan", "signup_date"]],
    on="user_id",
    how="left",
    validate="many_to_one",  # catch unexpected duplicate keys
)

# Multiple join keys
merged = pd.merge(orders, shipments, on=["order_id", "warehouse_id"], how="inner")

# Time-based merge: find the nearest preceding event
merged = pd.merge_asof(
    trades.sort_values("timestamp"),
    quotes.sort_values("timestamp"),
    on="timestamp",
    by="ticker",
    direction="backward",
)
```

### Groupby, Aggregation, and Pivot Tables

```python
# Multiple aggregations per column
summary = (
    df.groupby("region")
    .agg(
        total_revenue=("amount", "sum"),
        avg_order=("amount", "mean"),
        order_count=("amount", "count"),
        unique_users=("user_id", "nunique"),
    )
    .sort_values("total_revenue", ascending=False)
)

# Groupby with transform (broadcasts result back to original shape)
df["pct_of_region"] = df["amount"] / df.groupby("region")["amount"].transform("sum")

# Pivot table
pivot = df.pivot_table(
    values="amount",
    index="region",
    columns=pd.Grouper(key="created_at", freq="ME"),
    aggfunc="sum",
    fill_value=0,
)
```

### Method Chaining

```python
# Chain operations for readable data pipelines
result = (
    pd.read_csv("orders.csv", parse_dates=["order_date"])
    .query("status == 'completed'")
    .assign(
        order_month=lambda x: x["order_date"].dt.to_period("M"),
        revenue=lambda x: x["quantity"] * x["unit_price"],
    )
    .groupby("order_month")
    .agg(total_revenue=("revenue", "sum"), orders=("order_id", "nunique"))
    .reset_index()
    .sort_values("order_month")
)
```

### Performance: Vectorization and Categorical Dtypes

```python
# BAD: row-by-row iteration
for i, row in df.iterrows():
    df.at[i, "discount"] = row["amount"] * 0.1 if row["is_member"] else 0

# GOOD: vectorized operation (100x+ faster)
df["discount"] = np.where(df["is_member"], df["amount"] * 0.1, 0)

# Categorical dtype for low-cardinality strings
df["country"] = df["country"].astype("category")
# Before: 800 MB -> After: 80 MB for 10M rows with 50 unique countries

# Use .values or .to_numpy() when feeding into NumPy/sklearn
X = df[["feature_a", "feature_b"]].to_numpy()
```

**Anti-patterns to avoid:**
- Using `iterrows()` or `apply(axis=1)` when a vectorized operation exists -- these are 100-1000x slower.
- Reading CSV without specifying `dtype` or `parse_dates` -- leads to silent type coercion bugs.
- Calling `.copy()` excessively -- pandas returns views where safe; only copy when you need to modify a slice independently.
- Ignoring the `validate` parameter in `pd.merge()` -- undetected many-to-many joins silently explode row counts.

---

## SQL for Analysis

### Analytical Queries with Window Functions

```sql
-- Running total and month-over-month growth
SELECT
  month, revenue,
  SUM(revenue) OVER (ORDER BY month) AS cumulative_rev,
  revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
  ROUND((revenue - LAG(revenue) OVER (ORDER BY month))::NUMERIC
    / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 1) AS mom_growth_pct
FROM monthly_revenue;

-- Revenue rank per region
SELECT region, DATE_TRUNC('month', order_date) AS month, SUM(revenue) AS rev,
  RANK() OVER (PARTITION BY region ORDER BY SUM(revenue) DESC) AS rev_rank
FROM orders GROUP BY region, DATE_TRUNC('month', order_date);
```

### Cohort Analysis

```sql
-- Monthly retention cohort: % of users acquired in month X active in month X+N
WITH user_cohort AS (
  SELECT user_id, DATE_TRUNC('month', MIN(event_date)) AS cohort_month
  FROM events GROUP BY user_id
),
activity AS (
  SELECT e.user_id, uc.cohort_month,
    DATE_TRUNC('month', e.event_date) AS activity_month
  FROM events e JOIN user_cohort uc ON e.user_id = uc.user_id
)
SELECT
  cohort_month,
  DATE_PART('month', AGE(activity_month, cohort_month))::INT AS months_since,
  COUNT(DISTINCT user_id) AS active_users
FROM activity
GROUP BY cohort_month, activity_month
ORDER BY cohort_month, months_since;
-- Compute retention_pct in the application layer: active_users / cohort_size * 100
```

### Funnel Queries

```sql
-- Conversion funnel using conditional aggregation (no self-joins needed)
SELECT
  COUNT(DISTINCT CASE WHEN event = 'signup' THEN user_id END) AS signups,
  COUNT(DISTINCT CASE WHEN event = 'activation' THEN user_id END) AS activated,
  COUNT(DISTINCT CASE WHEN event = 'purchase' THEN user_id END) AS purchased,
  ROUND(COUNT(DISTINCT CASE WHEN event = 'activation' THEN user_id END)::NUMERIC /
        NULLIF(COUNT(DISTINCT CASE WHEN event = 'signup' THEN user_id END), 0) * 100, 1) AS activation_pct
FROM events WHERE event_date >= '2025-01-01';
```

### Retention Queries

```sql
-- Week-over-week retention: % of users active in week N who return in week N+1
WITH weekly_active AS (
  SELECT user_id, DATE_TRUNC('week', event_date) AS active_week
  FROM events GROUP BY user_id, DATE_TRUNC('week', event_date)
)
SELECT
  w1.active_week AS week,
  COUNT(DISTINCT w1.user_id) AS users_this_week,
  COUNT(DISTINCT w2.user_id) AS returned_next_week,
  ROUND(COUNT(DISTINCT w2.user_id)::NUMERIC / COUNT(DISTINCT w1.user_id) * 100, 1) AS retention_pct
FROM weekly_active w1
LEFT JOIN weekly_active w2
  ON w1.user_id = w2.user_id AND w2.active_week = w1.active_week + INTERVAL '1 week'
GROUP BY w1.active_week
ORDER BY w1.active_week;
```

### Time-Series Aggregation

```sql
-- Daily revenue with 7-day moving average (gap-filled with generate_series)
WITH daily AS (
  SELECT ds.day, COALESCE(SUM(o.revenue), 0) AS revenue
  FROM generate_series('2025-01-01'::date, CURRENT_DATE, '1 day') ds(day)
  LEFT JOIN orders o ON o.order_date::date = ds.day
  GROUP BY ds.day
)
SELECT day, revenue,
  AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma_7d,
  SUM(revenue) OVER (ORDER BY day) AS cumulative
FROM daily ORDER BY day;
```

**Anti-patterns to avoid:**
- Using `DISTINCT` inside `COUNT()` without understanding the performance cost -- it requires sorting or hashing the full result set.
- Writing funnel queries as sequential self-joins instead of conditional aggregation -- `COUNT(DISTINCT CASE WHEN ...)` is far more efficient.
- Forgetting to gap-fill time series with `generate_series` -- missing days produce misleading charts.

---

## Statistical Analysis

### Descriptive Statistics

```python
import numpy as np
import pandas as pd

# Summary stats with extended percentiles
print(df["revenue"].describe(percentiles=[0.25, 0.5, 0.75, 0.9, 0.95, 0.99]))
print(f"Skew: {df['revenue'].skew():.2f}, Kurtosis: {df['revenue'].kurtosis():.2f}")

# IQR-based outlier detection
Q1, Q3 = df["revenue"].quantile([0.25, 0.75])
IQR = Q3 - Q1
outliers = df[(df["revenue"] < Q1 - 1.5 * IQR) | (df["revenue"] > Q3 + 1.5 * IQR)]
print(f"Outliers: {len(outliers)} / {len(df)} ({len(outliers)/len(df)*100:.1f}%)")
```

### Hypothesis Testing

```python
from scipy import stats

# Two-sample t-test: is the mean revenue different between groups?
group_a = df[df["variant"] == "control"]["revenue"]
group_b = df[df["variant"] == "treatment"]["revenue"]

t_stat, p_value = stats.ttest_ind(group_a, group_b, equal_var=False)  # Welch's t-test
print(f"t={t_stat:.3f}, p={p_value:.4f}")
if p_value < 0.05:
    print("Reject H0: means are significantly different")

# Chi-square test: is conversion rate independent of traffic source?
contingency = pd.crosstab(df["source"], df["converted"])
chi2, p_value, dof, expected = stats.chi2_contingency(contingency)
print(f"chi2={chi2:.3f}, p={p_value:.4f}, dof={dof}")

# Mann-Whitney U test (non-parametric, for skewed distributions)
u_stat, p_value = stats.mannwhitneyu(group_a, group_b, alternative="two-sided")
print(f"U={u_stat:.0f}, p={p_value:.4f}")
```

### Confidence Intervals

```python
from scipy import stats
import numpy as np

# CI for a mean (95%, t-distribution)
data = df["revenue"].values
mean, se, n = np.mean(data), stats.sem(data), len(data)
ci = stats.t.interval(0.95, df=n - 1, loc=mean, scale=se)
print(f"Mean: {mean:.2f}, 95% CI: [{ci[0]:.2f}, {ci[1]:.2f}]")

# CI for a proportion (e.g., conversion rate)
p_hat = df["converted"].mean()
se_prop = np.sqrt(p_hat * (1 - p_hat) / len(df))
print(f"Rate: {p_hat:.4f}, 95% CI: [{p_hat - 1.96*se_prop:.4f}, {p_hat + 1.96*se_prop:.4f}]")

# Bootstrap CI (distribution-free, works for any statistic)
def bootstrap_ci(data, stat_func=np.mean, n_boot=10_000, ci=0.95):
    boots = [stat_func(np.random.choice(data, size=len(data), replace=True))
             for _ in range(n_boot)]
    alpha = (1 - ci) / 2
    return np.percentile(boots, [alpha * 100, (1 - alpha) * 100])

print(f"Bootstrap 95% CI: {bootstrap_ci(df['revenue'].values)}")
```

### Correlation and Regression

```python
from scipy import stats

# Pearson (linear) vs Spearman (monotonic, robust to outliers)
r, p = stats.pearsonr(df["ad_spend"], df["revenue"])
rho, p = stats.spearmanr(df["ad_spend"], df["revenue"])

# Simple linear regression
slope, intercept, r_val, p_val, se = stats.linregress(df["ad_spend"], df["revenue"])
print(f"revenue = {slope:.2f} * ad_spend + {intercept:.2f}, R2={r_val**2:.3f}")

# Multiple regression with statsmodels
import statsmodels.api as sm
X = sm.add_constant(df[["ad_spend", "email_opens", "sessions"]])
model = sm.OLS(df["revenue"], X).fit()
print(model.summary())
```

**Anti-patterns to avoid:**
- Using Pearson correlation on non-linear or heavily skewed data -- use Spearman or transform first.
- Interpreting p < 0.05 as "the effect is large" -- statistical significance is not practical significance. Always report effect sizes.
- Running multiple hypothesis tests without correction (Bonferroni or Benjamini-Hochberg) -- inflates false positive rate.
- Confusing correlation with causation -- correlation measures association, not causal direction.

---

## Data Visualization

### Matplotlib Fundamentals

```python
import matplotlib.pyplot as plt
import matplotlib.dates as mdates

plt.style.use("seaborn-v0_8-whitegrid")
fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(df["date"], df["revenue"], color="#2563eb", linewidth=2, label="Revenue")
ax.fill_between(df["date"], df["revenue"], alpha=0.1, color="#2563eb")
ax.set_title("Monthly Revenue", fontsize=16, fontweight="bold")
ax.set_ylabel("Revenue ($)")
ax.xaxis.set_major_formatter(mdates.DateFormatter("%b %Y"))
ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, _: f"${x:,.0f}"))
fig.autofmt_xdate()
fig.tight_layout()
fig.savefig("revenue.png", dpi=150, bbox_inches="tight")
```

### Seaborn for Statistical Plots

```python
import seaborn as sns

fig, axes = plt.subplots(1, 3, figsize=(15, 5))
sns.histplot(data=df, x="revenue", hue="plan", kde=True, ax=axes[0])
sns.boxplot(data=df, x="plan", y="revenue", ax=axes[1])
sns.violinplot(data=df, x="plan", y="revenue", inner="quartile", ax=axes[2])
fig.tight_layout()

# Correlation heatmap
corr = df[["revenue", "sessions", "ad_spend", "support_tickets"]].corr()
sns.heatmap(corr, annot=True, fmt=".2f", cmap="RdBu_r", center=0)
```

### Plotly for Interactive Charts

```python
import plotly.express as px
import plotly.graph_objects as go

# Interactive line chart
fig = px.line(monthly_df, x="month", y="revenue", color="region",
              title="Monthly Revenue by Region")
fig.update_layout(hovermode="x unified")
fig.show()

# Funnel chart
fig = go.Figure(go.Funnel(
    y=["Visited", "Signed Up", "Activated", "Subscribed"],
    x=[10000, 4200, 2800, 1100], textinfo="value+percent initial"))
fig.show()

# Cohort heatmap
fig = px.imshow(cohort_pivot, color_continuous_scale="Blues",
                labels=dict(x="Months Since Signup", y="Cohort", color="Retention %"),
                text_auto=".0f", title="Monthly Retention Cohort")
fig.show()
```

### Chart Selection Guide

| Data Question | Chart Type | Library |
|---|---|---|
| Trend over time | Line chart | matplotlib / plotly |
| Distribution shape | Histogram + KDE | seaborn |
| Group comparison | Box plot / violin | seaborn |
| Correlation matrix | Annotated heatmap | seaborn |
| Part-of-whole | Stacked bar / treemap | plotly |
| Funnel progression | Funnel chart | plotly |
| Geographic data | Choropleth map | plotly |
| Two-variable relationship | Scatter plot | seaborn / plotly |
| Cohort retention | Heatmap (months x cohort) | plotly / seaborn |
| KPI dashboard | Indicator + sparkline | plotly |

### Storytelling with Data Principles

1. **Lead with the insight, not the chart** -- title should state the conclusion ("Revenue grew 23% QoQ" not "Q3 Revenue").
2. **Remove clutter** -- no gridlines, no borders, no 3D, no decorative elements that don't encode data.
3. **Use color intentionally** -- one highlight color for the key series, gray for context. Never rainbow palettes for sequential data.
4. **Annotate directly** -- label data points on the chart instead of relying on legends.
5. **Order matters** -- sort bar charts by value, not alphabetically. Time series go left-to-right.

**Anti-patterns to avoid:**
- Pie charts for more than 3-4 categories -- human eyes compare angles poorly. Use horizontal bar charts.
- Dual y-axes -- they mislead because the scale relationship is arbitrary. Use two separate charts or normalize.
- Truncated y-axes on bar charts -- start bars at zero; truncation exaggerates differences.
- Default rainbow color palettes -- they are not perceptually uniform and fail for colorblind readers.

---

## A/B Test Analysis

### Experiment Design

```python
# Define experiment parameters before running
experiment = {
    "name": "checkout_button_color",
    "hypothesis": "Green CTA increases conversion rate vs blue",
    "primary_metric": "checkout_conversion_rate",
    "guardrail_metrics": ["revenue_per_user", "bounce_rate"],
    "traffic_split": 0.5,  # 50/50
    "min_detectable_effect": 0.02,  # 2 percentage points
    "significance_level": 0.05,  # alpha
    "power": 0.80,  # 1 - beta
    "duration_weeks": 2,
}
```

### Sample Size Calculation

```python
from scipy import stats
import numpy as np

def sample_size_proportion(
    baseline_rate: float,
    mde: float,  # minimum detectable effect (absolute)
    alpha: float = 0.05,
    power: float = 0.80,
) -> int:
    """Required sample size per variant for a two-proportion z-test."""
    p1 = baseline_rate
    p2 = baseline_rate + mde
    pooled_p = (p1 + p2) / 2
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)
    n = ((z_alpha * np.sqrt(2 * pooled_p * (1 - pooled_p)) +
          z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2) / (mde ** 2)
    return int(np.ceil(n))

# Example: baseline conversion = 10%, want to detect a 2pp lift
n = sample_size_proportion(baseline_rate=0.10, mde=0.02)
print(f"Need {n:,} users per variant ({n * 2:,} total)")
# -> ~3,842 per variant (7,684 total)
```

### Statistical Significance

```python
from scipy import stats
import numpy as np

def ab_test_proportions(n_a: int, conv_a: int, n_b: int, conv_b: int, alpha: float = 0.05) -> dict:
    """Two-proportion z-test for A/B experiment."""
    p_a, p_b = conv_a / n_a, conv_b / n_b
    p_pool = (conv_a + conv_b) / (n_a + n_b)

    se = np.sqrt(p_pool * (1 - p_pool) * (1 / n_a + 1 / n_b))
    z_stat = (p_b - p_a) / se
    p_value = 2 * (1 - stats.norm.cdf(abs(z_stat)))

    # Confidence interval for the difference
    se_diff = np.sqrt(p_a * (1 - p_a) / n_a + p_b * (1 - p_b) / n_b)
    z_crit = stats.norm.ppf(1 - alpha / 2)
    ci = ((p_b - p_a) - z_crit * se_diff, (p_b - p_a) + z_crit * se_diff)

    return {
        "control_rate": p_a, "treatment_rate": p_b,
        "lift": (p_b - p_a) / p_a, "p_value": p_value,
        "ci_95": ci, "significant": p_value < alpha,
    }

result = ab_test_proportions(5000, 500, 5000, 560)
# control=0.100, treatment=0.112, lift=+12%, p=0.049
```

### Bayesian A/B Testing

```python
import numpy as np

def bayesian_ab_test(conv_a: int, n_a: int, conv_b: int, n_b: int, sims: int = 100_000) -> dict:
    """Bayesian A/B test using Beta-Binomial conjugate model."""
    samples_a = np.random.beta(1 + conv_a, 1 + n_a - conv_a, size=sims)
    samples_b = np.random.beta(1 + conv_b, 1 + n_b - conv_b, size=sims)

    lift = (samples_b - samples_a) / samples_a
    return {
        "prob_b_better": (samples_b > samples_a).mean(),
        "expected_lift": lift.mean(),
        "ci_95": tuple(np.percentile(lift, [2.5, 97.5])),
    }

result = bayesian_ab_test(500, 5000, 560, 5000)
# prob_b_better=0.975, expected_lift=+12.1%, CI=[+0.2%, +25.1%]
```

### Common Pitfalls

1. **Peeking** -- checking results before the experiment reaches required sample size inflates false positive rates. Use sequential testing (e.g., always-valid p-values) if you need early stopping.
2. **Multiple comparisons** -- testing 10 metrics at alpha=0.05 gives a ~40% chance of at least one false positive. Apply Bonferroni correction: `alpha_corrected = 0.05 / n_tests`.
3. **Sample ratio mismatch (SRM)** -- if the actual traffic split deviates from intended (e.g., 48/52 instead of 50/50), the randomization is broken. Always run an SRM check before analyzing results.
4. **Novelty/primacy effects** -- new UI elements get artificially high engagement initially. Run experiments for at least 2 full business cycles (typically 2 weeks).
5. **Survivorship bias** -- analyzing only users who completed the funnel ignores drop-offs. Use intent-to-treat analysis (include all randomized users).

**Anti-patterns to avoid:**
- Stopping an experiment early because "it looks significant" -- peeking at alpha=0.05 daily can push actual false positive rates above 30%.
- Using revenue as the only metric -- always include guardrail metrics (error rate, latency, bounce rate) to catch regressions.
- Running experiments on less than one full week -- day-of-week effects create bias.
- Skipping the SRM check -- if observed traffic split deviates from expected (chi-square test, p < 0.01), randomization is broken and results are invalid.

---

## Business Metrics

### SaaS Metrics: MRR, ARR, Churn, LTV, CAC

```python
import pandas as pd
import numpy as np

# MRR breakdown: new + expansion - contraction - churned = net new
def mrr_breakdown(subs: pd.DataFrame) -> pd.Series:
    """Input: DataFrame with [user_id, month, mrr, is_new, is_churned, prev_mrr]."""
    new = subs.loc[subs["is_new"], "mrr"].sum()
    expansion_mask = (~subs["is_new"]) & (subs["mrr"] > subs["prev_mrr"])
    expansion = (subs.loc[expansion_mask, "mrr"] - subs.loc[expansion_mask, "prev_mrr"]).sum()
    contraction_mask = (~subs["is_new"]) & (subs["mrr"] < subs["prev_mrr"]) & (~subs["is_churned"])
    contraction = (subs.loc[contraction_mask, "prev_mrr"] - subs.loc[contraction_mask, "mrr"]).sum()
    churned = subs.loc[subs["is_churned"], "prev_mrr"].sum()
    return pd.Series({
        "new_mrr": new, "expansion": expansion,
        "contraction": contraction, "churned": churned,
        "net_new": new + expansion - contraction - churned,
    })

# Core SaaS formulas
def churn_rate(start: int, churned: int) -> float:
    return churned / start if start > 0 else 0.0

def ltv(arpu: float, monthly_churn: float) -> float:
    """LTV = ARPU / churn_rate (assumes constant churn)."""
    return arpu / monthly_churn if monthly_churn > 0 else float("inf")

def cac_payback_months(cac: float, arpu: float, gross_margin: float) -> float:
    return cac / (arpu * gross_margin)

# LTV:CAC ratio (target: > 3:1)
customer_ltv = ltv(arpu=85, monthly_churn=0.03)
cac = 250
print(f"LTV: ${customer_ltv:.0f}, CAC: ${cac}, LTV:CAC = {customer_ltv/cac:.1f}:1")
```

### Cohort Analysis in Python

```python
def build_cohort_table(events: pd.DataFrame) -> pd.DataFrame:
    """Build retention cohort table from [user_id, event_date] DataFrame."""
    events = events.copy()
    events["event_month"] = events["event_date"].dt.to_period("M")
    cohorts = events.groupby("user_id")["event_month"].min().rename("cohort")
    events = events.merge(cohorts, on="user_id")
    events["period"] = (events["event_month"] - events["cohort"]).apply(lambda x: x.n)

    cohort_data = events.groupby(["cohort", "period"])["user_id"].nunique().reset_index(name="users")
    cohort_size = cohort_data[cohort_data["period"] == 0].set_index("cohort")["users"]
    pivot = cohort_data.pivot(index="cohort", columns="period", values="users")
    return (pivot.divide(cohort_size, axis=0) * 100).round(1)
```

### Funnel Analysis

```python
def funnel_analysis(events: pd.DataFrame, steps: list[str]) -> pd.DataFrame:
    """Compute conversion rates between ordered funnel steps."""
    data = [{"step": s, "users": events[events["event_name"] == s]["user_id"].nunique()}
            for s in steps]
    result = pd.DataFrame(data)
    result["step_conversion"] = result["users"] / result["users"].shift(1)
    result["overall_conversion"] = result["users"] / result["users"].iloc[0]
    return result

funnel = funnel_analysis(events, ["page_view", "signup", "activation", "purchase"])
# step        | users | step_conversion | overall_conversion
# page_view   | 50000 | NaN             | 1.000
# signup      | 12000 | 0.240           | 0.240
# activation  |  7200 | 0.600           | 0.144
# purchase    |  2880 | 0.400           | 0.058
```

### RFM Segmentation

```python
def rfm_segmentation(orders: pd.DataFrame, reference_date: pd.Timestamp) -> pd.DataFrame:
    """Segment customers by Recency, Frequency, Monetary value."""
    rfm = orders.groupby("customer_id").agg(
        recency=("order_date", lambda x: (reference_date - x.max()).days),
        frequency=("order_date", "count"),
        monetary=("revenue", "sum"),
    )

    # Score each dimension 1-5 using quintiles
    for col in ["recency", "frequency", "monetary"]:
        ascending = col == "recency"  # lower recency = better
        rfm[f"{col}_score"] = pd.qcut(
            rfm[col], q=5, labels=[5, 4, 3, 2, 1] if ascending else [1, 2, 3, 4, 5]
        )

    rfm["rfm_segment"] = (
        rfm["recency_score"].astype(str)
        + rfm["frequency_score"].astype(str)
        + rfm["monetary_score"].astype(str)
    )
    return rfm

# Typical segment interpretation:
# 555 = Champions | 544 = Loyal | 444 = Potential Loyalists
# 211 = At Risk   | 111 = Lost  | 155 = Can't Lose
```

### North Star Metric Framework

| Company Type | North Star Metric | Supporting Metrics |
|---|---|---|
| SaaS | Weekly Active Users (WAU) | Activation rate, feature adoption, session length |
| Marketplace | Gross Merchandise Value (GMV) | Listings, buyers, take rate, repeat purchase |
| Media/Content | Time Spent Reading | DAU, articles/session, scroll depth, return rate |
| Fintech | Assets Under Management (AUM) | Deposits, transactions, net inflows |
| E-commerce | Revenue per Visitor (RPV) | Conversion rate, AOV, sessions, return rate |

**Anti-patterns to avoid:**
- Tracking vanity metrics (total signups, page views) instead of activation and retention metrics.
- Computing LTV without accounting for gross margin -- LTV should be profit-based, not revenue-based.
- Using logo churn (% of customers) instead of revenue churn for B2B SaaS -- one enterprise customer churning is not the same as one free-tier user.
- Measuring churn monthly when contracts are annual -- use the appropriate time window.

---

## Dashboard Design

### Dashboard Layout Principles

1. **Inverted pyramid** -- most critical KPIs at the top in large tiles, supporting detail below, raw data at the bottom.
2. **5-second rule** -- a viewer should understand the current state within 5 seconds of looking at the dashboard.
3. **One question per dashboard** -- "How is revenue trending?" and "What is the support queue?" are two different dashboards.
4. **Consistent time frame** -- all charts on one dashboard should share the same date filter. Mixing "last 7 days" with "last 30 days" creates confusion.
5. **Progressive disclosure** -- summary tiles link to drill-down dashboards for investigation.

### KPI Tile Design

```
+---------------------------------------------------------+
|  MRR: $142,500 (+8.3%)  |  Churn: 3.2% (-0.4pp)       |
|  New Customers: 287 (+12%)  |  NPS: 62 (+3)            |
+---------------------------------------------------------+
|  [Revenue Trend - 30d]       |  [MRR Breakdown Bar]    |
+---------------------------------------------------------+
|  [Conversion Funnel]         |  [Top Plans Table]      |
+---------------------------------------------------------+
```

Each KPI tile shows: current value, comparison period (MoM/WoW), direction indicator (green/red/gray), and an inline sparkline.

### Drill-Down Patterns

```
Level 0 (Executive):  MRR = $142,500 (+8.3%)
Level 1 (Breakdown):  New $18.2k | Expansion $9.4k | Contraction -$2.1k | Churn -$7.6k
Level 2 (Detail):     Churned accounts table (account, plan, MRR lost, reason)
Level 3 (Record):     Individual account timeline and activity log
```

Each level answers a progressively deeper question. Dashboards should link levels so users self-serve from "what happened?" to "why?".

### Real-Time vs Batch Dashboards

| Dimension | Real-Time | Batch (Scheduled Refresh) |
|---|---|---|
| Data freshness | Seconds to minutes | Hourly / daily |
| Use case | Ops monitoring, live campaigns | Business review, board reports |
| Data source | Streaming (Kafka, Kinesis) | Data warehouse (BigQuery, Snowflake) |
| Compute cost | High (continuous queries) | Low (run once per refresh) |
| Accuracy | Approximate (late events) | Exact (full reprocessing) |
| When to choose | Alert-driven decisions, live systems | Strategic decisions, trend analysis |

### Tools Comparison

| Tool | Best For | Hosting | SQL Support | Cost |
|---|---|---|---|---|
| Metabase | Self-serve analytics, quick setup | Self-hosted / Cloud | Native | Free OSS / paid cloud |
| Apache Superset | Advanced SQL analysts, large scale | Self-hosted | Native | Free OSS |
| Looker (Google) | Enterprise, governed metrics layer | Cloud | LookML modeling | Enterprise pricing |
| Grafana | Infrastructure and real-time metrics | Self-hosted / Cloud | Plugin-based | Free OSS / paid cloud |
| Tableau | Executive dashboards, storytelling | Desktop / Cloud | Visual + custom SQL | Per-user license |
| Streamlit | Custom analyst tools, prototypes | Self-hosted / Cloud | Via Python | Free OSS / paid cloud |

### Building a Metrics Layer

```yaml
# Define metrics once, reuse across dashboards (dbt metrics / Metriql style)
metrics:
  mrr:
    description: Monthly Recurring Revenue
    sql: SUM(mrr_amount)
    timestamp: month
    dimensions: [plan, region, acquisition_channel]
    filters: {status: active}

  churn_rate:
    description: Percentage of customers lost in the period
    sql: COUNT(CASE WHEN churned THEN 1 END)::FLOAT / COUNT(*)
    timestamp: month
    dimensions: [plan, region]
```

When every dashboard references the same metric definition, you eliminate conflicting numbers. Use dbt metrics, Metriql, or Looker's LookML to enforce this at the query layer.

**Anti-patterns to avoid:**
- Dashboards with more than 8-10 charts -- information overload reduces comprehension. Split into focused dashboards.
- Mixing operational and strategic metrics -- ops dashboards need real-time refresh; strategic dashboards need curated, slower cadence.
- Not defining metric calculations centrally -- when MRR is calculated differently on three dashboards, trust collapses.
- Missing date filters and clear "as of" timestamps -- viewers cannot assess data freshness.

---

## Common Patterns Quick Reference

| Pattern | When to Use | Key Tool / Library |
|---|---|---|
| `pd.read_parquet()` | Loading analytical datasets | `pandas` / `pyarrow` |
| `df.groupby().agg()` | Multi-metric summaries | `pandas` |
| `pd.merge_asof()` | Time-aligned joins (trades + quotes) | `pandas` |
| Method chaining | Readable transformation pipelines | `pandas` |
| Categorical dtype | Low-cardinality columns (save 90% memory) | `pandas` |
| Window functions (SQL) | Rankings, running totals, MoM growth | SQL (any dialect) |
| `generate_series` + LEFT JOIN | Gap-filling time series | PostgreSQL |
| Conditional aggregation | Funnel queries without self-joins | SQL |
| `scipy.stats.ttest_ind` | Two-group mean comparison | `scipy` |
| `scipy.stats.chi2_contingency` | Categorical independence test | `scipy` |
| Bootstrap CI | Distribution-free confidence interval | `numpy` |
| `statsmodels.OLS` | Multiple linear regression | `statsmodels` |
| `sns.heatmap` | Correlation matrix visualization | `seaborn` |
| `px.imshow` | Interactive cohort heatmap | `plotly` |
| `px.funnel` | Conversion funnel visualization | `plotly` |
| Two-proportion z-test | A/B test for conversion rates | `scipy` / custom |
| Beta-Binomial model | Bayesian A/B test | `numpy` / `scipy` |
| Sample size calculator | Pre-experiment power analysis | `scipy` |
| MRR waterfall | SaaS revenue decomposition | `pandas` / custom |
| RFM segmentation | Customer value segmentation | `pandas` |
| Cohort retention table | User retention over time | `pandas` / SQL |
| North Star + supporting metrics | Metric framework selection | framework |
| Inverted pyramid layout | Dashboard information hierarchy | design principle |
| Central metrics layer | Consistent metric definitions | dbt / Metriql / custom |
