# Telco Learning Day — Genie space (manual setup)

Use this in the **Databricks workspace** after **`01_generate_dataset.py`**, **`02_uc_documentation_and_constraints.py`**, and **`03_metric_view.py`** have run successfully.

**Unity Catalog:** `workspace.telco`  
**Metric views:** `telco_monthly_revenue_and_usage`, `telco_support_experience`, `telco_network_reliability`

For **programmatic import** (JSON + API) see [`04_genie_space.md`](04_genie_space.md).

---

## 1. Create the space

1. Sidebar → **Genie** → **New**.

---

## 2. Configure → Data

1. Switch the filter to `All` and browse to `workspace` then `telco`.
2. Add **data objects** (≤30 total; start focused — e.g. **three metric views only**, or MVs + `customers` if you need churn without writing SQL from scratch).

| Asset | Type | Suggested use |
|-------|------|----------------|
| `workspace.telco.telco_monthly_revenue_and_usage` | Metric view | ARPU, plan mix, regional usage, fees, overage |
| `workspace.telco.telco_support_experience` | Metric view | NPS, tickets, resolution time, CX by region |
| `workspace.telco.telco_network_reliability` | Metric view | Outages / maintenance by location |
| `workspace.telco.customers` | Table (optional) | Churn counts, subscriber status — only if Genie needs raw churn without composing from MVs |

**Recommendation:** Attach **all three metric views** first. Add **`customers`** only if demo questions need churn metrics not expressible from the metric views alone.

3. **Create**, then open **Configure** before sharing widely.
4. The space name, warehouse, etc. can be updated using the `About this space` edit option.
5. **Warehouse:** Pro or Serverless SQL warehouse (**CAN USE** required) — same tier you used for metric views (DBR 17.3+).

---

## 3. Instructions (text)

Paste into **Configure → Instructions**:

```text
You are helping analysts explore a synthetic Australian telco dataset (Jan–Mar 2026).

Definitions:
- ARPU (average revenue per user): average monthly plan fee plus overage charges per customer-month in the usage metric view. Prefer MEASURE(arpu_aud) from telco_monthly_revenue_and_usage when grouping by plan or region.
- Churn: subscribers with status = 'Churned' on the customers table; compare to Active. Regional areas often show higher churn than metro in this dataset.
- NPS: score 0–10 on support tickets; lower often correlates with churned accounts and longer resolution times.
- Severity on tickets: Low, Medium, High, Critical.
- Network events are outages and maintenance by city/state; they are not joined to individual subscribers in this synthetic set.

Prefer metric views for aggregates: use MEASURE(...) names defined on each metric view. When users ask about churn rate or subscriber counts by region, use the customers table or compose from available dimensions.

Currency is AUD unless stated otherwise.
```

---

## 4. SQL Queries (certified / trusted answers)

Add under **Configure → SQL Queries** add Example queries (titles = typical user phrasing). Adjust if your space uses only a subset of data objects.

**Finance / marketing — What is ARPU by plan?**

```sql
SELECT
  plan_name,
  MEASURE(arpu_aud) AS arpu_aud
FROM workspace.telco.telco_monthly_revenue_and_usage
GROUP BY plan_name
ORDER BY plan_name;
```

Usage Guidance: Use MEASURE(arpu_aud) from telco_monthly_revenue_and_usage for average revenue per customer-month by plan.

**Finance — What are total monthly plan fees and overage by region type?**

```sql
SELECT
  region_type,
  MEASURE(total_monthly_plan_aud) AS plan_fees_aud,
  MEASURE(total_overage_aud) AS overage_aud
FROM workspace.telco.telco_monthly_revenue_and_usage
GROUP BY region_type;
```

Usage Guidance - Finance view: compare Metro vs Regional fees and overage.

**CX — Which ticket categories have the most tickets and what is average NPS?**

```sql
SELECT
  ticket_category,
  MEASURE(ticket_count) AS tickets,
  MEASURE(avg_nps) AS avg_nps
FROM workspace.telco.telco_support_experience
GROUP BY ticket_category
ORDER BY tickets DESC;
```

Usage Guidance - CX: support experience metric view.

**CX — What is average resolution time in hours by metro vs regional?**

```sql
SELECT
  region_type,
  MEASURE(avg_resolution_hours) AS avg_resolution_hours
FROM workspace.telco.telco_support_experience
GROUP BY region_type;
```

Usage Guidance - CX: resolution time by region_type.

**Network — How many network events by event type and region?**

```sql
SELECT
  region_type,
  event_type,
  MEASURE(event_count) AS events,
  MEASURE(avg_duration_min) AS avg_duration_min
FROM workspace.telco.telco_network_reliability
GROUP BY region_type, event_type
ORDER BY region_type, events DESC;
```

Usage Guidance: Network reliability metric view.

**Churn (attach `customers` if you need this in-genie) - What is churn rate by region?**

```sql
SELECT
  region_type,
  COUNT(*) AS customers,
  SUM(CASE WHEN status = 'Churned' THEN 1 ELSE 0 END) AS churned,
  ROUND(100.0 * SUM(CASE WHEN status = 'Churned' THEN 1 ELSE 0 END) / COUNT(*), 1) AS churn_rate_pct
FROM workspace.telco.customers
GROUP BY region_type
ORDER BY region_type;
```

Usage Guidance - Churn requires the customers table; not in metric views.

---

**This is more than enough to have a working Genie Space.**

---

## 5. Demo questions (four pillars)

Use for live demo and benchmarks ([benchmarks](https://docs.databricks.com/aws/en/genie/benchmarks)).

| Pillar | Example natural-language questions |
|--------|-----------------------------------|
| **Finance** | What is ARPU by plan? Where are we losing most revenue to overage? |
| **Marketing** | How does prepaid vs postpaid mix look in the usage view? Compare metro vs regional data usage. |
| **CX** | Which ticket categories have the worst NPS? Is resolution time higher in regional areas? |
| **Network** | How many network events hit regional vs metro? Which event type is most common? |

---

## 6. Dry-run checklist

- [ ] Warehouse selected and you have **CAN USE**.
- [ ] **SELECT** on all attached UC objects.
- [ ] Instructions saved; no contradiction with metric view display names.
- [ ] At least **6–8** example SQL queries saved (mix of MEASURE queries + optional churn SQL).
- [ ] Run each **benchmark** question; fix instructions or examples if SQL is wrong.
- [ ] Note **time-to-answer** for the Learn session (target within your 10-minute demo block).

---

## 7. References

- [Set up and manage a Genie space](https://docs.databricks.com/aws/en/genie/set-up)
- [Curate an effective Genie space](https://docs.databricks.com/aws/en/genie/best-practices)
- [Query metric views](https://docs.databricks.com/aws/en/business-semantics/metric-views/query)
