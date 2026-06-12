# Insight data spec examples

Reference outlines following the skill output template. Use for calibration when drafting new specs.

**Insight types:** For a taxonomy of analysis patterns (snapshot, historical chart, period aggregation, top N, recurring detection, allocation) and how they map to these examples, see [insight-types.md](../insight-types.md).

**Before drafting a new insight:** ask clarifying questions (timeframe, account scope, filters, etc.) — see skill workflow. These examples document **resolved** decisions; they are not templates with open questions.

**To add a new insight:** after clarifying with the user, save the spec in the domain folder that matches the insight (`net-worth/`, `cash-flow/`, `investment-account/`, or `alerts/`), then add a row to the index below and to the calibration table in [SKILL.md](../SKILL.md). Every finalized insight becomes a reference for the skill.

## Folders

| Category | Folder | Examples |
|---|---|---|
| Net worth | [net-worth/](net-worth/) | Balance-sheet snapshots, assets vs liabilities; shared [net-worth-core.md](net-worth/net-worth-core.md) partial for snapshot + chart |
| Cash flow | [cash-flow/](cash-flow/) | Spending, income, transfers over time; shared [cash-flow-core.md](cash-flow/cash-flow-core.md) joined transaction table with `account_type` |
| Investment account | [investment-account/](investment-account/) | Holdings, allocation, recurring investment patterns |
| Alerts | [alerts/](alerts/) | Actionable flags and threshold-based recommendations |

## Index

### Net worth

| Insight | File | Analysis pattern |
|---|---|---|
| Net worth snapshot | [net-worth-snapshot.md](net-worth/net-worth-snapshot.md) | Snapshot |
| Net worth balance chart | [net-worth-balance-chart.md](net-worth/net-worth-balance-chart.md) | Historical chart |
| Assets / liabilities bar | [assets-liabilities-bar.md](net-worth/assets-liabilities-bar.md) | Snapshot + composition |
| Cash account detail | [cash-account-detail.md](net-worth/cash-account-detail.md) | Snapshot + composite |
| Overview dashboard | [overview-dashboard.md](net-worth/overview-dashboard.md) | Composite |

### Cash flow

| Insight | File | Analysis pattern |
|---|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](cash-flow/monthly-spending-by-category.md) | Period aggregation |
| Recurring transactions | [recurring-transactions.md](cash-flow/recurring-transactions.md) | Recurring detection |
| Top 5 biggest purchases | [top-5-biggest-purchases.md](cash-flow/top-5-biggest-purchases.md) | Top N ranking |
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](cash-flow/cash-inflow-outflow-chart.md) | Period aggregation |

### Investment account

| Insight | File | Analysis pattern |
|---|---|---|
| Top 5 holdings | [top-5-holdings.md](investment-account/top-5-holdings.md) | Snapshot + top N |
| Asset allocation | [asset-allocation.md](investment-account/asset-allocation.md) | Snapshot + composition |
| Investment accounts by institution | [investment-accounts-by-institution.md](investment-account/investment-accounts-by-institution.md) | Snapshot |
| Recurring investments | [recurring-investments.md](investment-account/recurring-investments.md) | Recurring detection |
| Investment performance chart | [investment-performance-chart.md](investment-account/investment-performance-chart.md) | Historical chart |
| Investment account detail | [investment-account-detail.md](investment-account/investment-account-detail.md) | Composite |

### Alerts

| Insight | File | Analysis pattern |
|---|---|---|
| Excess checking cash | [excess-checking-cash.md](alerts/excess-checking-cash.md) | Snapshot |
| Subscription price increase | [subscription-price-increase.md](alerts/subscription-price-increase.md) | Recurring detection + change |
| Late paycheck | [late-paycheck.md](alerts/late-paycheck.md) | Recurring detection + lateness |

## Shared partials (not insights)

| Partial | Folder | Used by |
|---|---|---|
| Net worth core | [net-worth-core.md](net-worth/net-worth-core.md) | [Net worth snapshot](net-worth/net-worth-snapshot.md), [Net worth balance chart](net-worth/net-worth-balance-chart.md), [Assets / liabilities bar](net-worth/assets-liabilities-bar.md), [Overview dashboard](net-worth/overview-dashboard.md) |
| Cash flow core | [cash-flow-core.md](cash-flow/cash-flow-core.md) | [Monthly spending by category](cash-flow/monthly-spending-by-category.md), [Cash inflow and outflow chart](cash-flow/cash-inflow-outflow-chart.md), [Recurring transactions](cash-flow/recurring-transactions.md), [Top 5 biggest purchases](cash-flow/top-5-biggest-purchases.md), [Late paycheck](alerts/late-paycheck.md) |

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
