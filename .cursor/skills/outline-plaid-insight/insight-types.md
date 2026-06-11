# Insight types catalog

A taxonomy of insight **analysis patterns** we can build from Plaid datatables, grounded in the finalized examples in [examples/](examples/README.md). Use this when scoping a new insight: pick a pattern, then adapt domain (net worth, cash flow, investments), timeframe, and UI.

For step-by-step spec authoring, see [SKILL.md](SKILL.md). For UI mapping, see [ui-output-options.md](ui-output-options.md).

---

## How patterns relate to domains

| Domain | Folder | Typical questions |
|---|---|---|
| **Net worth** | [net-worth/](examples/net-worth/) | What do I own vs owe? How has net worth changed? |
| **Cash flow** | [cash-flow/](examples/cash-flow/) | Where does money go? What repeats every month? |
| **Investment account** | [investment-account/](examples/investment-account/) | What are my largest positions? How is the portfolio split? What do I invest on a schedule? |
| **Alerts** | [alerts/](examples/alerts/) | Should I take action? Am I above or below a recommended threshold? |

One insight usually combines **one analysis pattern** + **one domain** + **one UI pattern**.

---

## Browse by UI output type

For full build specs, see [ui-output-options.md](ui-output-options.md). Example specs remain in domain folders; the index below groups by how each insight renders.

| UI type | Insights |
|---|---|
| [Nested list](ui-output-options.md#nested-list) | Net worth snapshot, Top 5 holdings, Asset allocation, Investment accounts by institution |
| [Flat table](ui-output-options.md#flat-table) | Monthly spending by category, Recurring spending, Top 5 biggest purchases, Recurring investments |
| [Line chart](ui-output-options.md#line-chart) | Net worth balance chart, Investment performance chart |
| [Combo line and bar chart](ui-output-options.md#combo-line-and-bar-chart) | Cash inflow and outflow chart |
| [Insight card](ui-output-options.md#insight-card) | Excess checking cash, Subscription price increase |

| Insight | Domain | File | UI type |
|---|---|---|---|
| Net worth snapshot | Net worth | [net-worth-snapshot.md](examples/net-worth/net-worth-snapshot.md) | Nested list |
| Top 5 holdings | Investment | [top-5-holdings.md](examples/investment-account/top-5-holdings.md) | Nested list |
| Asset allocation | Investment | [asset-allocation.md](examples/investment-account/asset-allocation.md) | Nested list |
| Investment accounts by institution | Investment | [investment-accounts-by-institution.md](examples/investment-account/investment-accounts-by-institution.md) | Nested list |
| Monthly spending by category | Cash flow | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) | Flat table |
| Recurring spending | Cash flow | [recurring-spending.md](examples/cash-flow/recurring-spending.md) | Flat table |
| Top 5 biggest purchases | Cash flow | [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md) | Flat table |
| Recurring investments | Investment | [recurring-investments.md](examples/investment-account/recurring-investments.md) | Flat table |
| Net worth balance chart | Net worth | [net-worth-balance-chart.md](examples/net-worth/net-worth-balance-chart.md) | Line chart |
| Investment performance chart | Investment | [investment-performance-chart.md](examples/investment-account/investment-performance-chart.md) | Line chart |
| Cash inflow and outflow chart | Cash flow | [cash-inflow-outflow-chart.md](examples/cash-flow/cash-inflow-outflow-chart.md) | Combo line + bar chart |
| Excess checking cash | Alerts | [excess-checking-cash.md](examples/alerts/excess-checking-cash.md) | Insight card |
| Subscription price increase | Alerts | [subscription-price-increase.md](examples/alerts/subscription-price-increase.md) | Insight card |

---

## Analysis patterns

### 1. Snapshot (point-in-time)

**Question:** What is the value *right now* (latest sync)?

**Data shape:** Single `as_of` timestamp; detail rows at one moment; rollups derived from detail (not computed separately).

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_accounts` (balances), `plaid_investment_holdings` + `plaid_investment_securities` (positions) |
| **Time** | Latest snapshot per user (`synced_at = MAX`) — via [net-worth-core](examples/net-worth/net-worth-core.md) `resolve_accounts_as_of` at `MAX(synced_at)` |
| **Aggregation** | Per account, per security, or portfolio-wide sum |
| **Output** | Scalar totals + array of detail rows |
| **UI** | [Nested list](ui-output-options.md#net-worth--nested-list) (hierarchy with balances at each level) |
| **Shared core** | [net-worth-core.md](examples/net-worth/net-worth-core.md) — `compute_net_worth(..., include_groups = true)` |

**Examples**

| Insight | File |
|---|---|
| Net worth snapshot | [net-worth-snapshot.md](examples/net-worth/net-worth-snapshot.md) |
| Top 5 holdings | [top-5-holdings.md](examples/investment-account/top-5-holdings.md) |
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Investment accounts by institution | [investment-accounts-by-institution.md](examples/investment-account/investment-accounts-by-institution.md) |

**Variants you can build**

- Single-account balance snapshot (checking + savings roll-up)
- Excess checking cash vs recommended minimum — [excess-checking-cash.md](examples/alerts/excess-checking-cash.md)
- Liabilities-only snapshot (credit cards + loans)
- Full holdings list (same as top holdings without the top-5 cap)
- Cash vs invested split across investment accounts

---

### 2. Historical balance chart (time series)

**Question:** How did a balance or total change over a selected period?

**Data shape:** Ordered `{ date, value }` points; optional min/max for chart axis; parameterized `timeframe`.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_accounts` (retain all historical `synced_at` rows, not latest-only) |
| **Time** | Calendar day series; point-in-time account state per day (carry forward between syncs) |
| **Parameters** | `trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |
| **Output** | `points[]`, `window_start`, `window_end`, optional `value_min` / `value_max` |
| **UI** | [Line chart](ui-output-options.md#net-worth-balance-chart--line-chart) |
| **Shared core** | [net-worth-core.md](examples/net-worth/net-worth-core.md) — daily `resolve_accounts_as_of` + `compute_net_worth(..., include_groups = false)` per day |

**Examples**

| Insight | File |
|---|---|
| Net worth balance chart | [net-worth-balance-chart.md](examples/net-worth/net-worth-balance-chart.md) |

**Variants you can build**

- Total assets or total liabilities over time (same machinery, different roll-up)
- Single-account balance history
- Investment portfolio value over time (holdings history if retained per `synced_at`)
- Spending or income cumulative line (transaction-derived, not balance snapshots)

---

### 3. Period aggregation (bucket + dimension)

**Question:** How much happened in each time bucket, broken down by a dimension (category, merchant, account)?

**Data shape:** Flat rows keyed by `period` + `dimension`; period totals derived from rows.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_transactions` |
| **Time** | Calendar month (or week/quarter) from transaction `date` |
| **Dimension** | `personal_finance_category_primary`, merchant, account, etc. |
| **Filters** | Exclude pending, removed, transfers/income/loan payments as needed; net refunds into category |
| **Output** | `rows[]` + `period_totals[]` (or equivalent); or `months[]` for chart series |
| **UI** | [Flat table](ui-output-options.md#monthly-spending-by-category--flat-table), [Combo line + bar chart](ui-output-options.md#cash-inflow-outflow-chart--combo-line-and-bar-chart) |
| **Shared core** | [cash-flow-core.md](examples/cash-flow/cash-flow-core.md) — transactions joined to `account_type`; callers own scope, timeframe, and eligibility |

**Examples**

| Insight | File |
|---|---|
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) |
| Cash inflow and outflow chart | [cash-inflow-outflow-chart.md](examples/cash-flow/cash-inflow-outflow-chart.md) |

**Variants you can build**

- Spending by **detailed** category (`personal_finance_category_detailed`)
- Income by month and source category
- Spend by merchant (top merchants per month)
- Investment buys/sells by month and security type
- Comparison windows (this month vs prior month) — extra date filters, same pattern

---

### 4. Composition / allocation breakdown

**Question:** What share of a total belongs to each class or bucket?

**Data shape:** Classes with `value`, `percent`, and optional holdings drill-down; `total` = sum of parts; percents from total.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | Holdings + securities; may require extension tables for fund decomposition |
| **Classification** | Rules on security `type`, ISIN, enrichment weights, or category codes |
| **Output** | `asset_classes[]` or category shares; `total_*_value`; handle `unclassified` explicitly |
| **UI** | [Nested list](ui-output-options.md#asset-allocation--nested-list) with % / $ toggle at class level |

**Examples**

| Insight | File |
|---|---|
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) *(share-of-month view)* |

**Variants you can build**

- Sector or geography allocation (needs enrichment beyond Plaid)
- Account-type mix (% in 401k vs brokerage vs IRA)
- Spend mix pie for a single month (same data as monthly-by-category, different UI)
- Liability mix (mortgage vs credit card vs student loan)

**Schema note:** ETF/mutual fund allocation requires `security_asset_allocation` (or similar) — not in raw Plaid schema. See [asset-allocation.md](examples/investment-account/asset-allocation.md).

---

### 5. Top N ranking

**Question:** What are the largest items by value or amount?

**Data shape:** Sorted list capped at N; often includes per-account or per-source breakdown under each ranked item.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | Holdings (`institution_value`), transactions (summed `amount`), or aggregated buckets |
| **Sort** | Descending by value or amount |
| **Limit** | Top 5, top 10, configurable `N` |
| **Output** | Ranked array with `rank`, `total_value`, nested `accounts[]` or similar |
| **UI** | [Nested list](ui-output-options.md#top-5-holdings--nested-list) |

**Examples**

| Insight | File |
|---|---|
| Top 5 holdings | [top-5-holdings.md](examples/investment-account/top-5-holdings.md) |

**Variants you can build**

- Top spending categories (month or trailing 3 months)
- Top individual purchases (trailing 30 days) — [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md)
- Top merchants by spend
- Top gainers/losers *(blocked or approximate without cost basis / historical prices)*
- Largest recurring bills (combine with recurring detection, then rank by `typical_amount`)

---

### 6. Recurring pattern detection

**Question:** What charges or contributions happen on a predictable schedule?

**Data shape:** Flat list of detected patterns: merchant or security, frequency, typical amount, last occurrence; 6-month detection window is the current convention.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_transactions` (spend) or `plaid_investment_transactions` (invest) |
| **Detection** | Group by merchant or account+security; ≥2 occurrences; median interval → frequency bucket; amount consistency (±20% for spend); active filter from `last_date` |
| **Frequencies** | Weekly, biweekly, semi-monthly, monthly, quarterly, annual |
| **Output** | `recurrences[]`, `window_start`, `window_end` |
| **UI** | [Flat table](ui-output-options.md#recurring-spending--flat-table) |
| **Shared core** | [cash-flow-core.md](examples/cash-flow/cash-flow-core.md) — joined transaction table; callers apply scope, timeframe, and eligibility ([recurring spending](examples/cash-flow/recurring-spending.md), [late paycheck](examples/alerts/late-paycheck.md)) |

**Examples**

| Insight | File |
|---|---|
| Recurring spending | [recurring-spending.md](examples/cash-flow/recurring-spending.md) |
| Recurring investments | [recurring-investments.md](examples/investment-account/recurring-investments.md) |

**Variants you can build**

- Recurring income (payroll detection — filter to `INCOME` category, inflows)
- Late paycheck alert — [late-paycheck.md](examples/alerts/late-paycheck.md)
- Subscriptions only (filter categories or merchant tags)
- Subscription price increase alert — [subscription-price-increase.md](examples/alerts/subscription-price-increase.md)
- Estimated monthly recurring total (rollup row derived from `recurrences[]`)
- Dormant recurrences (failed active filter — “used to pay X”)

**Schema note:** Plaid has no recurring-transaction flag in the datatables; patterns are **inferred**. Document limitations in the spec (missed merges, variable bills, merchant name drift).

---

## UI patterns × analysis patterns

| UI pattern | Best for analysis patterns |
|---|---|
| **Nested list** | Snapshot, composition/allocation, top N with drill-down |
| **Line chart** | Historical balance / value time series |
| **Combo line + bar chart** | Period aggregation with inflow/outflow bars and a net trend line |
| **Flat table** | Period aggregation, recurring lists, simple ranked tables without nesting |
| **Insight card** | Snapshot alerts with hero metric and optional account breakdown |

Full UI specs: [ui-output-options.md](ui-output-options.md).

---

## Quick reference: all finalized examples

Grouped by UI output type. Full index: [examples/README.md](examples/README.md).

| Insight | Domain | Analysis pattern | UI |
|---|---|---|---|
| Net worth snapshot | Net worth | Snapshot | Nested list |
| Top 5 holdings | Investment | Snapshot + top N | Nested list |
| Asset allocation | Investment | Snapshot + composition | Nested list |
| Investment accounts by institution | Investment | Snapshot | Nested list |
| Monthly spending by category | Cash flow | Period aggregation | Flat table |
| Top 5 biggest purchases | Cash flow | Top N ranking | Flat table |
| Recurring spending | Cash flow | Recurring detection | Flat table |
| Recurring investments | Investment | Recurring detection | Flat table |
| Net worth balance chart | Net worth | Historical chart | Line chart |
| Investment performance chart | Investment | Historical chart | Line chart |
| Cash inflow and outflow chart | Cash flow | Period aggregation | Combo line + bar chart |
| Excess checking cash | Alerts | Snapshot | Insight card |
| Subscription price increase | Alerts | Recurring detection + change | Insight card |
| Late paycheck | Alerts | Recurring detection + lateness | Insight card |

---

## Choosing a pattern for a new insight

1. **What is the user asking?** — “right now”, “over time”, “how much per month”, “what repeats”, “what’s biggest”, or “how is it split”.
2. **What data exists?** — Balances need `plaid_accounts` history for charts; spend needs `plaid_transactions`; positions need holdings + securities.
3. **Snapshot or history?** — Latest `synced_at` vs point-in-time per day vs transaction `date` buckets.
4. **Detail + rollup rule** — Totals always derived from detail rows ([rollup convention](SKILL.md#rollup-convention) in SKILL.md).
5. **Pick UI** — Match shape to [ui-output-options.md](ui-output-options.md) or define a new section there.

When the insight is finalized, add a spec under [examples/](examples/README.md) and a row to the calibration table in [SKILL.md](SKILL.md).
