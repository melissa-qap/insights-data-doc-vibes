---
name: outline-plaid-insight
description: >-
  Outlines data inputs, analysis, and outputs for a financial insight using
  Plaid datatables from plaid-api-schema.md. Use when the user describes an
  insight, asks for data requirements, input/output mapping, or how to build
  an insight from Plaid data.
---

# Outline Plaid Insight Data Spec

Produce a concise data spec for a financial insight: required Plaid tables/columns, calculation steps, and data output fields.

## Resources

| File | Purpose |
|---|---|
| [plaid-api-schema.md](plaid-api-schema.md) | Hybrid datatable + Plaid API spec reference (Account APIs overview, endpoint fields) — read before drafting |
| [insight-types.md](insight-types.md) | Taxonomy of insight patterns (snapshot, chart, recurring, top N, etc.) |
| [examples/](examples/README.md) | One `.md` per finalized insight — add every new insight here |
| [design-api](../design-api/SKILL.md) | API design — sync/read routes, request/response contracts, client composition |

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
| **Null / missing data** | Exclude, fallback, zero | Filter rules |

**Defaults** (only if user declines to specify): latest snapshot for balances; `trailing_1y` for charts; current calendar month for spending; exclude `pending = true` and `removed = true`; primary category for spend grouping.

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

**Parameters** (when the insight accepts caller input):

| Parameter | Type | Default | Notes / options |
|---|---|---|---|
| `param` | type | default | Allowed values, validation, or behavior notes |

### Calculation / analysis

1. **Step name**
   - Detail
   - Detail
2. **Next step**
   - Detail
3. **Format output**
   - Apply [output formatting](#output-formatting) to all monetary and fraction-percent fields

### Data output

**Formatting:** Dollar fields — 2 dp; fraction percent fields — 3 dp ([output formatting](#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `field` | type | ... |
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

## Output formatting

Round only when serializing output — keep full precision during calculation.

| Output type | Rule | Notes |
|---|---|---|
| Dollar / monetary | 2 decimal places | Any field representing USD (balances, amounts, values, returns in $) |
| Fraction percent | 3 decimal places | Fields stored as 0–1 (e.g. `0.035` = 3.5%); not display `%` strings |
| Excluded | No rounding | Counts, ranks, config constants, share `quantity`, dates |

Apply to every emitted number in **Data output** tables. When drafting new specs, add a final calculation step: **Format output** — apply output formatting to all monetary and fraction-percent fields.

## Rollup convention

When output has both detail rows and totals (e.g. accounts + net worth, categories + month total), **derive totals from detail rows** — do not compute totals independently.

## Gap handling

If data is not in the schema, state **Not available in current Plaid schema**, suggest a proxy, or mark the insight blocked.

## Calibration examples

Grouped by domain. Full index: [examples/README.md](examples/README.md).

### Net worth

| Insight | File | Analysis pattern |
|---|---|---|
| Cash account detail | [cash-account-detail.md](examples/net-worth/cash-account-detail.md) | Snapshot + composite |
| Overview dashboard | [overview-dashboard.md](examples/net-worth/overview-dashboard.md) | Composite |

Net worth chart, snapshot, and bar APIs → [design-api net worth APIs](../design-api/examples/net-worth/net-worth-apis.md).

### Cash flow

| Insight | File | Analysis pattern |
|---|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) | Period aggregation + Top N |
| Recurring transactions (V1) | [recurring-transactions-v1.md](examples/cash-flow/recurring-transactions-v1.md) | Recurring detection (Plaid API) |
| Recurring transactions (V2) | [recurring-transactions.md](examples/cash-flow/recurring-transactions.md) | Recurring detection (inferred) |
| Top 5 biggest purchases | [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md) | Top N ranking |
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](examples/cash-flow/cash-inflow-outflow-chart.md) | Period aggregation |

### Investment account

| Insight | File | Analysis pattern |
|---|---|---|
| Holdings by value | [holdings-by-value.md](examples/investment-account/holdings-by-value.md) | Snapshot |
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) | Snapshot + composition |
| Investment accounts by institution | [investment-accounts-by-institution.md](examples/investment-account/investment-accounts-by-institution.md) | Snapshot |
| Recurring investments | [recurring-investments.md](examples/investment-account/recurring-investments.md) | Recurring detection |
| Investment performance chart | [investment-performance-chart.md](examples/investment-account/investment-performance-chart.md) | Historical chart (wrapper) |
| Investment account detail | [investment-account-detail.md](examples/investment-account/investment-account-detail.md) | Composite |

### Alerts

| Insight | File | Analysis pattern |
|---|---|---|
| Excess checking cash | [excess-checking-cash.md](examples/alerts/excess-checking-cash.md) | Snapshot |
| Subscription price increase | [subscription-price-increase.md](examples/alerts/subscription-price-increase.md) | Recurring detection + change |
| Late paycheck | [late-paycheck.md](examples/alerts/late-paycheck.md) | Recurring detection + lateness |

Match structure and tone; do not copy logic for a different insight.
