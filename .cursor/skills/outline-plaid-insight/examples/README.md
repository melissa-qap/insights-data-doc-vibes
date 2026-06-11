# Insight data spec examples

Reference outlines following the skill output template. Use for calibration when drafting new specs.

**Insight types:** For a taxonomy of analysis patterns (snapshot, historical chart, period aggregation, top N, recurring detection, allocation) and how they map to these examples, see [insight-types.md](../insight-types.md).

**UI output types:** For how data output maps to presentation patterns, see [ui-output-options.md](../ui-output-options.md) — organized by UI type (nested list, flat table, line chart, combo chart, insight card).

**Before drafting a new insight:** ask clarifying questions (timeframe, account scope, filters, UI pattern, etc.) — see skill workflow. These examples document **resolved** decisions; they are not templates with open questions.

**To add a new insight:** after clarifying with the user, save the spec in the domain folder that matches the insight (`net-worth/`, `cash-flow/`, `investment-account/`, or `alerts/`), then add a row under the matching UI type in the index below, and add a row to the calibration table in [SKILL.md](../SKILL.md). Every finalized insight becomes a reference for the skill.

## Folders

| Category | Folder | Examples |
|---|---|---|
| Net worth | [net-worth/](net-worth/) | Balance-sheet snapshots, assets vs liabilities; shared [net-worth-core.md](net-worth/net-worth-core.md) partial for snapshot + chart |
| Cash flow | [cash-flow/](cash-flow/) | Spending, income, transfers over time; shared [cash-flow-core.md](cash-flow/cash-flow-core.md) joined transaction table with `account_type` |
| Investment account | [investment-account/](investment-account/) | Holdings, allocation, recurring investment patterns |
| Alerts | [alerts/](alerts/) | Actionable flags and threshold-based recommendations |

## Index

### Nested list

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Net worth snapshot | Net worth | [net-worth-snapshot.md](net-worth/net-worth-snapshot.md) | [Nested list](../ui-output-options.md#net-worth--nested-list) |
| Top 5 holdings | Investment account | [top-5-holdings.md](investment-account/top-5-holdings.md) | [Nested list](../ui-output-options.md#top-5-holdings--nested-list) |
| Asset allocation | Investment account | [asset-allocation.md](investment-account/asset-allocation.md) | [Nested list](../ui-output-options.md#asset-allocation--nested-list) |
| Investment accounts by institution | Investment account | [investment-accounts-by-institution.md](investment-account/investment-accounts-by-institution.md) | [Nested list](../ui-output-options.md#investment-accounts-by-institution--nested-list) |

### Flat table

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Monthly spending by category | Cash flow | [monthly-spending-by-category.md](cash-flow/monthly-spending-by-category.md) | [Flat table](../ui-output-options.md#monthly-spending-by-category--flat-table) |
| Recurring spending | Cash flow | [recurring-spending.md](cash-flow/recurring-spending.md) | [Flat table](../ui-output-options.md#recurring-spending--flat-table) |
| Top 5 biggest purchases | Cash flow | [top-5-biggest-purchases.md](cash-flow/top-5-biggest-purchases.md) | [Flat table](../ui-output-options.md#top-5-biggest-purchases--flat-table) |
| Recurring investments | Investment account | [recurring-investments.md](investment-account/recurring-investments.md) | [Flat table](../ui-output-options.md#recurring-investments--flat-table) |
| Cash account detail | Net worth | [cash-account-detail.md](net-worth/cash-account-detail.md) | [Account detail — flat table](../ui-output-options.md#account-detail--flat-table) |

### Line chart

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Net worth balance chart | Net worth | [net-worth-balance-chart.md](net-worth/net-worth-balance-chart.md) | [Line chart](../ui-output-options.md#net-worth-balance-chart--line-chart) |
| Investment performance chart | Investment account | [investment-performance-chart.md](investment-account/investment-performance-chart.md) | [Line chart](../ui-output-options.md#investment-performance-chart--line-chart) |

### Combo line and bar chart

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Cash inflow and outflow chart | Cash flow | [cash-inflow-outflow-chart.md](cash-flow/cash-inflow-outflow-chart.md) | [Combo line + bar chart](../ui-output-options.md#cash-inflow-outflow-chart--combo-line-and-bar-chart) |

### Stacked bar

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Assets / liabilities bar | Net worth | [assets-liabilities-bar.md](net-worth/assets-liabilities-bar.md) | [Stacked bar](../ui-output-options.md#assets--liabilities-bar--stacked-bar) |

### Insight card

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Excess checking cash | Alerts | [excess-checking-cash.md](alerts/excess-checking-cash.md) | [Insight card](../ui-output-options.md#excess-checking-cash--insight-card) |
| Subscription price increase | Alerts | [subscription-price-increase.md](alerts/subscription-price-increase.md) | [Insight card](../ui-output-options.md#subscription-price-increase--insight-card) |
| Late paycheck | Alerts | [late-paycheck.md](alerts/late-paycheck.md) | [Insight card](../ui-output-options.md#late-paycheck--insight-card) |

### Composite

| Insight | Domain | File | UI spec |
|---|---|---|---|
| Overview dashboard | Net worth | [overview-dashboard.md](net-worth/overview-dashboard.md) | [Composite](../ui-output-options.md#overview-dashboard--composite) |
| Investment account detail | Investment account | [investment-account-detail.md](investment-account/investment-account-detail.md) | [Composite](../ui-output-options.md#investment-account-detail--composite) |

## Shared partials (not insights)

| Partial | Folder | Used by |
|---|---|---|
| Net worth core | [net-worth-core.md](net-worth/net-worth-core.md) | [Net worth snapshot](net-worth/net-worth-snapshot.md), [Net worth balance chart](net-worth/net-worth-balance-chart.md), [Assets / liabilities bar](net-worth/assets-liabilities-bar.md), [Overview dashboard](net-worth/overview-dashboard.md) |
| Cash flow core | [cash-flow-core.md](cash-flow/cash-flow-core.md) | [Monthly spending by category](cash-flow/monthly-spending-by-category.md), [Cash inflow and outflow chart](cash-flow/cash-inflow-outflow-chart.md), [Recurring spending](cash-flow/recurring-spending.md), [Top 5 biggest purchases](cash-flow/top-5-biggest-purchases.md), [Late paycheck](alerts/late-paycheck.md) |

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

### UI output

**Pattern:** [Link to ui-output-options.md or define new pattern]
```
