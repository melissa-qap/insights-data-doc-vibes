---
name: outline-plaid-insight
description: >-
  Outlines data inputs, analysis, and outputs for a financial insight using
  Plaid datatables from plaid-api-schema.md. Use when the user describes an
  insight, asks for data requirements, input/output mapping, or how to build
  an insight from Plaid data.
---

# Outline Plaid Insight Data Spec

Produce a concise data spec for a financial insight: required Plaid tables/columns, calculation steps, data output fields, and UI presentation.

## Resources

| File | Purpose |
|---|---|
| [plaid-api-schema.md](plaid-api-schema.md) | Hybrid datatable + Plaid API spec reference (Account APIs overview, endpoint fields) — read before drafting |
| [insight-types.md](insight-types.md) | Taxonomy of insight patterns (snapshot, chart, recurring, top N, etc.) |
| [examples/](examples/README.md) | One `.md` per finalized insight — add every new insight here |
| [ui-output-options.md](ui-output-options.md) | UI output types (nested list, flat table, line chart, etc.) — grouped by type |

## Workflow

1. **Clarify before drafting** — Use `AskQuestion` (or ask conversationally) to resolve ambiguities **before** writing the spec. Ask 1–2 questions at a time. Skip questions already answered. Do not produce the full doc until timeframe and other data-shaping decisions are clear, or the user says to proceed with defaults.

2. **Parse the insight** — What question does it answer? Apply clarified answers to time window, granularity, and snapshot vs trend.

3. **Read schema** — Load [plaid-api-schema.md](plaid-api-schema.md). Use only tables and columns that exist there. Cite names exactly (e.g. `plaid_transactions.personal_finance_category_primary`).

4. **Draft the spec** — Use the [output template](#output-template) below. Record resolved decisions inline; no open questions in the final doc.

5. **Validate** — Every column must exist in the schema. Flag gaps as **Not available in current Plaid schema** with the closest proxy or mark blocked.

### Clarification checklist

Ask only what is ambiguous:

| Topic | Examples | Affects |
|---|---|---|
| **Timeframe** | Latest snapshot, calendar month, trailing months, date range | Row filters, snapshot vs history |
| **Account scope** | All accounts, investment only, depository + credit | `plaid_accounts.type`, which tables |
| **Aggregation** | Per user, account, security, category | Group-by keys, rollups |
| **Inclusions / exclusions** | Categories, pending, transfers, loan payments | Filters, allowlists |
| **Amount handling** | Outflows only, net refunds, gross | `amount` sign rules |
| **Ranking / limits** | Top 5, top 10, none | Sort, row cap |
| **Comparison** | vs prior month/year, none | Extra date windows |
| **UI pattern** | Nested list, flat table | Link or add [ui-output-options.md](ui-output-options.md) |
| **Null / missing data** | Exclude, fallback, zero | Filter rules |

**Defaults** (only if user declines to specify): latest snapshot for balances; current calendar month for spending; exclude `pending = true` and `removed = true`; primary category for spend grouping.

## Output template

**Every finalized insight** must be saved as `examples/{category}/{kebab-case-name}.md` where `{category}` is `net-worth`, `cash-flow`, `investment-account`, or `alerts`, indexed in [examples/README.md](examples/README.md), and listed in [Calibration examples](#calibration-examples) below.

```markdown
# [Insight name]

### Description
[1–2 sentences]

### Required input data

#### `table_name`

| Column | Description |
|---|---|
| `col` | What this column provides |

**Input:** Short filters, scope, and joins.

### Calculation / analysis

1. **Step name**
   - Detail
   - Detail
2. **Next step**
   - Detail

### Data output

| Field | Type | Description |
|---|---|---|
| `field` | type | ... |

### UI output

**Pattern:** [Link to ui-output-options.md section or define new pattern]
```

Keep specs under ~60 lines unless the insight is complex.

## Schema rules

| Rule | Notes |
|---|---|
| Query datatables only | Never call Plaid APIs at runtime |
| Scope by `user_id` | All queries |
| `synced_at` | Current snapshot vs point-in-time on balance/holdings tables |
| Pending transactions | Exclude `pending = true` from analysis |
| Removed transactions | Exclude `removed = true` |
| `amount` sign | Positive = money out, negative = money in |
| Investment holdings | Join `plaid_investment_holdings` ↔ `plaid_investment_securities` on `security_id` + matching `synced_at` |
| Liabilities | Join liability tables to `plaid_accounts` on `account_id` |
| Display names | Prefer `merchant_name` over `name` for transactions |
| Nullable fields | Document exclude vs fallback (`balances_available`, `cost_basis`, `institution_value`) |

## Rollup convention

When output has both detail rows and totals (e.g. accounts + net worth, categories + month total), **derive totals from detail rows** — do not compute totals independently.

## Gap handling

If data is not in the schema, state **Not available in current Plaid schema**, suggest a proxy, or mark the insight blocked.

## Calibration examples

Grouped by UI output type. Full index: [examples/README.md](examples/README.md).

### Nested list

| Insight | File |
|---|---|
| Net worth snapshot | [net-worth-snapshot.md](examples/net-worth/net-worth-snapshot.md) |
| Top 5 holdings | [top-5-holdings.md](examples/investment-account/top-5-holdings.md) |
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Investment accounts by institution | [investment-accounts-by-institution.md](examples/investment-account/investment-accounts-by-institution.md) |

### Flat table

| Insight | File |
|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) |
| Recurring spending | [recurring-spending.md](examples/cash-flow/recurring-spending.md) |
| Top 5 biggest purchases | [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md) |
| Recurring investments | [recurring-investments.md](examples/investment-account/recurring-investments.md) |
| Cash account detail | [cash-account-detail.md](examples/net-worth/cash-account-detail.md) |

### Line chart

| Insight | File |
|---|---|
| Net worth balance chart | [net-worth-balance-chart.md](examples/net-worth/net-worth-balance-chart.md) |
| Investment performance chart | [investment-performance-chart.md](examples/investment-account/investment-performance-chart.md) |

### Composite

| Insight | File |
|---|---|
| Overview dashboard | [overview-dashboard.md](examples/net-worth/overview-dashboard.md) |
| Investment account detail | [investment-account-detail.md](examples/investment-account/investment-account-detail.md) |

### Combo line and bar chart

| Insight | File |
|---|---|
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](examples/cash-flow/cash-inflow-outflow-chart.md) |

### Stacked bar

| Insight | File |
|---|---|
| Assets / liabilities bar | [assets-liabilities-bar.md](examples/net-worth/assets-liabilities-bar.md) |

### Insight card

| Insight | File |
|---|---|
| Excess checking cash | [excess-checking-cash.md](examples/alerts/excess-checking-cash.md) |
| Subscription price increase | [subscription-price-increase.md](examples/alerts/subscription-price-increase.md) |
| Late paycheck | [late-paycheck.md](examples/alerts/late-paycheck.md) |

Match structure and tone; do not copy logic for a different insight.
