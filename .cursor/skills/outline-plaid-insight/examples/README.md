# Insight data spec examples

Reference outlines following the skill output template. Use for calibration when drafting new specs.

**Insight types:** For a taxonomy of analysis patterns (snapshot, historical chart, period aggregation, top N, recurring detection, allocation) and how they map to these examples, see [insight-types.md](../insight-types.md).

**Before drafting a new insight:** ask clarifying questions (timeframe, account scope, filters, UI pattern, etc.) — see skill workflow. These examples document **resolved** decisions; they are not templates with open questions.

**To add a new insight:** after clarifying with the user, save the spec in the folder that matches the insight type (kebab-case filename), add a row to the index below, and add a row to the calibration table in [SKILL.md](../SKILL.md). Every finalized insight becomes a reference for the skill.

## Folders

| Category | Folder | Examples |
|---|---|---|
| Net worth | [net-worth/](net-worth/) | Balance-sheet snapshots, assets vs liabilities |
| Cash flow | [cash-flow/](cash-flow/) | Spending, income, transfers over time |
| Investment account | [investment-account/](investment-account/) | Holdings, allocation, recurring investment patterns |

## Index

### Net worth

| Insight | File | UI pattern |
|---|---|---|
| Net worth snapshot | [net-worth-snapshot.md](net-worth/net-worth-snapshot.md) | [Nested list](../ui-output-options.md#net-worth--nested-list) |
| Net worth balance chart | [net-worth-balance-chart.md](net-worth/net-worth-balance-chart.md) | [Line chart](../ui-output-options.md#net-worth-balance-chart--line-chart) |

### Cash flow

| Insight | File | UI pattern |
|---|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](cash-flow/monthly-spending-by-category.md) | [Flat table](../ui-output-options.md#monthly-spending-by-category--flat-table) |
| Recurring spending | [recurring-spending.md](cash-flow/recurring-spending.md) | [Flat table](../ui-output-options.md#recurring-spending--flat-table) |
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](cash-flow/cash-inflow-outflow-chart.md) | [Combo line + bar chart](../ui-output-options.md#cash-inflow-outflow-chart--combo-line-and-bar-chart) |

### Investment account

| Insight | File | UI pattern |
|---|---|---|
| Top 5 holdings | [top-5-holdings.md](investment-account/top-5-holdings.md) | [Nested list](../ui-output-options.md#top-5-holdings--nested-list) |
| Recurring investments | [recurring-investments.md](investment-account/recurring-investments.md) | [Flat table](../ui-output-options.md#recurring-investments--flat-table) |
| Asset allocation | [asset-allocation.md](investment-account/asset-allocation.md) | [Nested list](../ui-output-options.md#asset-allocation--nested-list) |
| Investment performance chart | [investment-performance-chart.md](investment-account/investment-performance-chart.md) | [Line chart](../ui-output-options.md#investment-performance-chart--line-chart) |
| Investment accounts by institution | [investment-accounts-by-institution.md](investment-account/investment-accounts-by-institution.md) | [Nested list](../ui-output-options.md#investment-accounts-by-institution--nested-list) |

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
1. ...

### Data output

| Field | Type | Description |
|---|---|---|
| ... | ... | ... |

### UI output

**Pattern:** [Link to ui-output-options.md or define new pattern]
```
