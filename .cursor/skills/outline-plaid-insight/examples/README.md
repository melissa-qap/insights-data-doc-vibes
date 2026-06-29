# Insight data spec examples

Reference outlines following the skill output template. Use for calibration when drafting new specs.

**Insight types:** For a taxonomy of analysis patterns (snapshot, historical chart, period aggregation, top N, recurring detection, allocation) and how they map to these examples, see [insight-types.md](../insight-types.md).

**Before drafting a new insight:** ask clarifying questions (timeframe, account scope, filters, etc.) — see skill workflow. These examples document **resolved** decisions; they are not templates with open questions.

**To add a new insight:** after clarifying with the user, save the spec in the domain folder that matches the insight (`net-worth/`, `cash-flow/`, `investment-account/`, or `alerts/`), then add a row to the index below and to the calibration table in [SKILL.md](../SKILL.md). Every finalized insight becomes a reference for the skill.

**API specs** (routes, sync/read contracts, client composition) live in the sibling [design-api](../../design-api/examples/README.md) skill — not here.

## Folders

| Category | Folder | Examples |
|---|---|---|
| Net worth | [net-worth/](net-worth/) | Shared [net-worth-core.md](net-worth/net-worth-core.md) partial; chart/snapshot/bar → [design-api net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md) |
| Cash flow | [cash-flow/](cash-flow/) | Spending, income, transfers over time; shared [cash-flow-core.md](cash-flow/cash-flow-core.md) joined transaction table with `account_type` → [design-api cash flow APIs](../../design-api/examples/cash-flow/cash-flow-apis.md) |
| Investment account | [investment-account/](investment-account/) | Holdings, allocation, recurring investment patterns → [design-api investment account APIs](../../design-api/examples/investment-account/investment-account-apis.md) |
| Alerts | [alerts/](alerts/) | Actionable flags and threshold-based recommendations |

## Index

### Net worth

| Insight | File | Analysis pattern |
|---|---|---|
| Cash account detail | [cash-account-detail.md](net-worth/cash-account-detail.md) | Snapshot + composite |
| Overview dashboard | [overview-dashboard.md](net-worth/overview-dashboard.md) | Composite |

Net worth read APIs → [design-api net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md).

### Cash flow

| Insight | File | Analysis pattern |
|---|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](cash-flow/monthly-spending-by-category.md) | Period aggregation + Top N |
| Recurring transactions (V1) | [recurring-transactions-v1.md](cash-flow/recurring-transactions-v1.md) | Recurring detection (Plaid API) |
| Recurring transactions (V2) | [recurring-transactions.md](cash-flow/recurring-transactions.md) | Recurring detection (inferred) |
| Top 5 biggest purchases | [top-5-biggest-purchases.md](cash-flow/top-5-biggest-purchases.md) | Top N ranking |
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](cash-flow/cash-inflow-outflow-chart.md) | Period aggregation |

Cash flow sync API → [design-api cash flow APIs](../../design-api/examples/cash-flow/cash-flow-apis.md). Read endpoints not yet defined.

### Investment account

| Insight | File | Analysis pattern |
|---|---|---|
| Holdings by value | [holdings-by-value.md](investment-account/holdings-by-value.md) | Snapshot |
| Asset allocation | [asset-allocation.md](investment-account/asset-allocation.md) | Snapshot + composition |
| Investment accounts by institution | [investment-accounts-by-institution.md](investment-account/investment-accounts-by-institution.md) | Snapshot |
| Recurring investments | [recurring-investments.md](investment-account/recurring-investments.md) | Recurring detection |
| Investment performance chart | [investment-performance-chart.md](investment-account/investment-performance-chart.md) | Historical chart (wrapper) |
| Investment account detail | [investment-account-detail.md](investment-account/investment-account-detail.md) | Composite |

Investment read APIs → [design-api investment account APIs](../../design-api/examples/investment-account/investment-account-apis.md#read-apis). Account list reuses `GET /v1/account-balance` (flat, no institution grouping); performance chart reuses `GET /v1/performance-history`.

### Alerts

| Insight | File | Analysis pattern |
|---|---|---|
| Excess checking cash | [excess-checking-cash.md](alerts/excess-checking-cash.md) | Snapshot |
| Subscription price increase | [subscription-price-increase.md](alerts/subscription-price-increase.md) | Recurring detection + change |
| Late paycheck | [late-paycheck.md](alerts/late-paycheck.md) | Recurring detection + lateness |

## Shared partials (not insights)

| Partial | Folder | Used by |
|---|---|---|
| Net worth core | [net-worth-core.md](net-worth/net-worth-core.md) | [Net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md), [Overview dashboard](net-worth/overview-dashboard.md), [Investment performance chart](investment-account/investment-performance-chart.md), [Investment account APIs](../../design-api/examples/investment-account/investment-account-apis.md) |
| Cash flow core | [cash-flow-core.md](cash-flow/cash-flow-core.md) | [Monthly spending by category](cash-flow/monthly-spending-by-category.md), [Cash inflow and outflow chart](cash-flow/cash-inflow-outflow-chart.md), [Recurring transactions (V1)](cash-flow/recurring-transactions-v1.md), [Recurring transactions (V2)](cash-flow/recurring-transactions.md), [Top 5 biggest purchases](cash-flow/top-5-biggest-purchases.md), [Late paycheck](alerts/late-paycheck.md) |

## Template

Copy this structure for new insight files:

```markdown
# [Insight name]

### Description
[1–2 sentences]

### Required input data

#### `table_name`

| Column | Description |
|---|---|
| `col` | ... |

**Input:** ...

### Calculation / analysis

1. **Step name**
   - Detail
   - Detail
2. **Next step**
   - Detail

### Data output

| Field | Type | Description |
|---|---|---|
| ... | ... | ... |
```
