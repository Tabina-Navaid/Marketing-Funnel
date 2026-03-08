# 📊 Marketing Funnel Analysis — EDA, LTV, CAC & ROI

A structured, end-to-end marketing analytics notebook built in Python. This project gives a growth/marketing team a **repeatable framework** to evaluate acquisition campaign performance across five channels, Organic, Paid Search, Paid Social, Referral, and Email.

---

## 🧩 Problem Statement

The marketing team had no repeatable, structured framework to evaluate acquisition campaigns. Specifically:

- No visibility into **which channels were actually profitable**
- No view of **where users were dropping off** in the funnel
- No measurement of **cost per acquired customer (CAC)** by channel
- No estimate of **long-term customer value (LTV)** by channel or signup cohort

---

## 📁 Repository Structure

```
├── Marketing_Funnel_EDA.ipynb   ← Main analysis notebook
├── marketing_funnel_dataset.csv                       ← Input dataset (add here)
└── README.md
```

---

## 🔬 Analysis Phases

The notebook is structured across **six sequential phases:**

1. **EDA** — data quality audit (missing values, zeros, skew, correlations)
2. **Funnel Analysis** — Impressions → Clicks → Signups → Activated → Converted
3. **Funnel by Channel** — separating high-volume vs. high-quality channels
4. **LTV** — Monthly Revenue × Months Active, by channel and signup cohort
5. **CAC** — Total Campaign Cost ÷ Converted Customers, by channel
6. **ROI** — (LTV − Cost) / Cost, profitability ranking by channel

---

## 🛠️ Libraries Used

| Library | Purpose |
|---------|---------|
| `pandas` | Data manipulation and aggregation |
| `numpy` | Numerical calculations, handling missing/infinite values |
| `matplotlib` | All visualisations — histograms, funnel charts, bar charts, line plots, heatmaps |

---

## 💻 Key Code Snippets

### Phase 1 — Missing Value Audit

```python
missing = df.isna().sum().sort_values(ascending=False)
missing_pct = (missing / len(df) * 100).round(2)
missing_table = pd.DataFrame({"missing_count": missing, "missing_pct": missing_pct})
```

```python
# Zero checks on impression and click fields
zero_checks = pd.DataFrame({
    "zero_count": (df[["impressions", "clicks"]].fillna(0) == 0).sum(),
    "zero_pct": ((df[["impressions", "clicks"]].fillna(0) == 0).sum() / len(df) * 100).round(2)
})
```

---

### Phase 2 — Funnel Construction & Conversion Rates

```python
funnel = pd.DataFrame({
    "Exposed (Impressions>0)": df["impressions"].fillna(0) > 0,
    "Clicked (Clicks>0)":      df["clicks"].fillna(0) > 0,
    "Signed Up":               df["signups"].fillna(0).astype(int) == 1,
    "Activated":               df["activated"].fillna(0).astype(int) == 1,
    "Converted":               df["converted"].fillna(0).astype(int) == 1,
})
funnel_counts = funnel.sum().astype(int)
```

```python
# Step-to-step conversion rates
rates = []
for i in range(1, len(counts)):
    prev, curr = counts[i-1], counts[i]
    rates.append(curr / prev if prev else np.nan)

funnel_rates = pd.DataFrame({
    "from_step": steps[:-1],
    "to_step":   steps[1:],
    "from_count": counts[:-1],
    "to_count":   counts[1:],
    "conversion_rate": np.round(rates, 4)
})
```

```python
# Centered horizontal bar funnel chart
left_offsets = (max_val - values) / 2.0
plt.barh(y_pos, values, left=left_offsets)
plt.gca().invert_yaxis()
plt.title("Marketing Funnel (Centered Bars)")
```

---

### Phase 3 — Funnel Breakdown by Channel

```python
def channel_funnel(group):
    f = pd.DataFrame({
        "Exposed":   group["impressions"].fillna(0) > 0,
        "Clicked":   group["clicks"].fillna(0) > 0,
        "Signed Up": group["signups"].fillna(0).astype(int) == 1,
        "Activated": group["activated"].fillna(0).astype(int) == 1,
        "Converted": group["converted"].fillna(0).astype(int) == 1,
    })
    return f.sum().astype(int)

channel_funnel_counts = df2.groupby("channel_clean").apply(channel_funnel)

# Exposed → Converted rate per channel
conv_by_channel = (
    channel_funnel_counts["Converted"] / channel_funnel_counts["Exposed"]
).replace([np.inf, -np.inf], np.nan)
```

---

### Phase 4 — LTV (Lifetime Value)

```python
# LTV proxy: Monthly Revenue × Months Active
ltv["monthly_revenue_clean"] = ltv["monthly_revenue"].fillna(0)
ltv["months_active_clean"]   = ltv["months_active"].fillna(0)
ltv["ltv_clean"] = ltv["monthly_revenue_clean"] * ltv["months_active_clean"]
```

```python
# LTV by channel
ltv_by_channel = (
    ltv.groupby("channel_clean")["ltv_clean"]
    .agg(["count", "mean", "median", "sum"])
    .sort_values("mean", ascending=False)
)
```

```python
# LTV by signup month cohort
ltv["signup_month"] = ltv["signup_date"].dt.to_period("M").astype(str)
cohort_ltv = ltv.groupby("signup_month")["ltv_clean"].mean().sort_index()
```

---

### Phase 5 — CAC (Customer Acquisition Cost)

```python
# CAC = Total Campaign Cost / Number of Converted Customers
cac["campaign_cost_clean"] = cac["campaign_cost"].fillna(0)

total_cost      = cac["campaign_cost_clean"].sum()
total_customers = cac["converted_int"].sum()
overall_cac     = total_cost / total_customers if total_customers else np.nan
```

```python
# CAC by channel
cost_by_channel = cac.groupby("channel_clean")["campaign_cost_clean"].sum()
cust_by_channel = cac.groupby("channel_clean")["converted_int"].sum()
cac_by_channel  = (cost_by_channel / cust_by_channel).replace([np.inf, -np.inf], np.nan).sort_values()
```

---

### Phase 6 — ROI (Return on Investment)

```python
# ROI = (LTV - Cost) / Cost
total_rev    = roi["ltv_clean"].sum()
total_cost   = roi["campaign_cost_clean"].sum()
overall_roi  = (total_rev - total_cost) / total_cost if total_cost else np.nan
```

```python
# ROI by channel
rev_by_channel  = roi.groupby("channel_clean")["ltv_clean"].sum()
cost_by_channel = roi.groupby("channel_clean")["campaign_cost_clean"].sum()
roi_by_channel  = (
    (rev_by_channel - cost_by_channel) / cost_by_channel
).replace([np.inf, -np.inf], np.nan).sort_values(ascending=False)
```

---

## 📈 Business Outcomes

- **Funnel** — exposed the exact step where user drop-off was most severe, giving the growth team a precise intervention point
- **Channel quality** — separated high-volume channels from high-conversion ones using the Exposed → Converted rate
- **LTV** — identified which channels attract the highest-value customers and how cohort quality shifted over signup months
- **CAC** — ranked channels by cost efficiency, highlighting the cheapest and most expensive acquisition sources
- **ROI** — delivered a direct profitability ranking across all five channels, enabling smarter budget reallocation

---

## 📸 Notebook Output 


### Funnel 
<img width="671" height="363" alt="image" src="https://github.com/user-attachments/assets/477becc3-e33b-4d3f-9379-049b3f458c90" />

### LTV by Channel
<img width="661" height="292" alt="image" src="https://github.com/user-attachments/assets/184f576d-5df7-463a-957d-3d27e91d0c95" />


### CAC by Channel
<img width="641" height="269" alt="image" src="https://github.com/user-attachments/assets/9edc6076-016d-4d2b-afd1-56ca1a041149" />


### ROI by Channel
<img width="664" height="276" alt="image" src="https://github.com/user-attachments/assets/71e6dab2-7c1b-414a-8eed-5902f46f0014" />



---

