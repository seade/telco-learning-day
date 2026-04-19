# Telco Genie Learning Day

An end-to-end workshop that generates a synthetic Australian telco dataset, enriches it with Unity Catalog documentation and constraints, builds metric views, and culminates in a configured Genie space for natural-language analytics.

> **Tip — Databricks Free Edition** provides everything you need to load and run this workshop: a serverless SQL warehouse, Unity Catalog, metric views, and Genie spaces — all at no cost. Sign up at [databricks.com/try](https://www.databricks.com/try) if you don't already have a workspace. The `workspace` schema used throughout is provided by default in Free Edition.

**Unity Catalog target:** `workspace.telco`

---

## Generated Data

Notebook `01` produces five tables in Unity Catalog modelling a simplified Australian telco:

| Table | Rows (approx) | Description |
|-------|---------------|-------------|
| `plans` | 6 | Reference dimension — six mobile plan tiers from Essential 30 GB (Prepaid, $35/mo) through Premium Unlimited (Postpaid, $150/mo). |
| `customers` | ~10,000 | Subscriber dimension — each customer has a state, city, metro/regional classification, signup date, plan, device type (4G/5G), and churn status. ~3% are flagged as Churned. |
| `usage` | ~30,000 | Monthly fact table (Jan–Mar 2026) — data consumed (GB), call minutes, SMS count, and overage charges per customer-month. Regional users tend to use slightly less data. |
| `support_tickets` | ~13,500 | CX fact table — ticket category, severity, channel, NPS score (0–10), and resolution time in hours. Churned customers generate more tickets with worse NPS. |
| `network_events` | ~200 | Infrastructure events (outages and maintenance) by city/state over the same three months. ~60% are in regional areas with longer durations. |

### Key data stories baked in

- **Regional vs metro gap** — regional subscribers show higher churn (~4% vs ~1.8%), longer support resolution times, lower NPS, and more network events.
- **Clear churn signals** — short tenure, prepaid plans, 4G devices, and high complaint frequency all correlate with churn.
- **Australian-flavoured** — real Australian cities, AUD currency, and plan names echoing local telco tiers.

---

## Notebooks

### 01 — Generate Dataset (`01_generate_dataset`)

Creates the five tables listed above using PySpark with deterministic random seeds. Steps:

1. Sets catalog and schema (`workspace.telco`).
2. Defines Australian states with metro and regional cities (65/35 split).
3. Builds the `plans` reference table (6 rows).
4. Generates ~10,000 `customers` with realistic churn probability influenced by region, tenure, plan type, and device.
5. Creates ~30,000 `usage` rows (3 months per active customer) with plan-aware data, calls, SMS, and overage.
6. Produces ~13,500 `support_tickets` with category, severity, NPS, and resolution hours skewed by churn status and region.
7. Generates ~200 `network_events` (outages and maintenance), biased toward regional areas.

All tables are written via `saveAsTable` (Delta, managed).

### 02 — UC Documentation & Constraints (`02_uc_documentation_and_constraints`)

Run after notebook 01. Enriches the raw tables so Genie (and other tools) can leverage semantics and join relationships:

1. **Table and column comments** — `COMMENT ON TABLE` and `COMMENT ON COLUMN` for every table and every column, describing business meaning and join keys.
2. **Primary keys** — informational `PRIMARY KEY` constraints on `plan_id`, `customer_id`, `(customer_id, month)`, `ticket_id`, and `event_id`. Columns are first set `NOT NULL`.
3. **Foreign keys** — informational `FOREIGN KEY` constraints linking `customers.plan_id → plans.plan_id`, `usage.customer_id → customers.customer_id`, and `support_tickets.customer_id → customers.customer_id`.
4. **Verification** — queries `information_schema.table_constraints` to confirm all PKs and FKs are in place.

Constraints are informational only (not engine-enforced).

### 03 — Metric Views (`03_metric_view`)

Run after notebook 02. Creates three metric views using YAML 1.1 definitions (requires Databricks Runtime 17.3+):

| Metric View | Grain | Key Measures |
|-------------|-------|-------------|
| `telco_monthly_revenue_and_usage` | Customer-month | ARPU (AUD), total plan fees, total overage, avg data usage (GB), avg call minutes |
| `telco_support_experience` | Ticket | Ticket count, average NPS, average resolution hours |
| `telco_network_reliability` | Event | Event count, average duration (minutes) |

Each metric view includes:
- **Dimensions** with `display_name`, `comment`, and `synonyms` for Genie discoverability.
- **Measures** using `MEASURE(...)` aggregation syntax with `format` hints (e.g. `$#,##0.00` for currency).
- **Joins** to dimension tables (e.g. usage → customers → plans via snowflake-path column references).

A smoke query at the end validates ARPU by plan.

---

## Genie Space — Manual Creation (Step 04)

After all three notebooks have run, create the Genie space via the Databricks UI:

1. **Create** — Sidebar → Genie → New. Name it (e.g. "Telco Genie demo"). Select a Pro or Serverless SQL warehouse (CAN USE required, DBR 17.3+).

2. **Add data objects** — In Configure → Data, attach the three metric views (and optionally the `customers` table for direct churn queries):
   - `workspace.telco.telco_monthly_revenue_and_usage`
   - `workspace.telco.telco_support_experience`
   - `workspace.telco.telco_network_reliability`
   - `workspace.telco.customers` *(optional)*

3. **Add instructions** — Paste domain instructions into Configure → Instructions covering ARPU definition, churn logic, NPS interpretation, severity levels, network event scope, and currency (AUD).

4. **Add example SQL** — Add 6–8 certified queries under Configure → Example SQL covering ARPU by plan, revenue by region, NPS by ticket category, resolution time by region, network events by type, and churn rate by region.

5. **Dry-run** — Verify warehouse access, SELECT permissions on all objects, instruction consistency with metric view display names, and benchmark each demo question for correctness and response time.

See `notebooks/04_genie_space_manual.md` for the full step-by-step guide, ready-to-paste instructions text, and all example SQL queries.

---

## Project Structure

```
.
├── README.md                          ← This file
├── databricks.yml                     ← Databricks Asset Bundle config
└── notebooks/
    ├── 01_generate_dataset            ← Synthetic data generation
    ├── 02_uc_documentation_and_constraints  ← Comments, PKs, FKs
    ├── 03_metric_view                 ← YAML metric view creation
    └── 04_genie_space_manual.md       ← Manual UI setup guide
```

---

## Quick Start

1. Run **`01_generate_dataset`** to create the five tables.
2. Run **`02_uc_documentation_and_constraints`** to add comments and constraints.
3. Run **`03_metric_view`** to create the three metric views.
4. Follow **`04_genie_space_manual.md`** to build the Genie space in the UI.
5. Ask Genie questions like *"What is ARPU by plan?"* or *"Is resolution time higher in regional areas?"*
