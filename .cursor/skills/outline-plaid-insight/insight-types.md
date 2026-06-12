# Insight types catalog

A taxonomy of insight **analysis patterns** we can build from Plaid datatables, grounded in the finalized examples in [examples/](examples/README.md). Use this when scoping a new insight: pick a pattern, then adapt domain (net worth, cash flow, investments) and timeframe.

For step-by-step spec authoring, see [SKILL.md](SKILL.md).

---

## How patterns relate to domains

| Domain | Folder | Typical questions |
|---|---|---|
| **Net worth** | [net-worth/](examples/net-worth/) | What do I own vs owe? How has net worth changed? |
| **Cash flow** | [cash-flow/](examples/cash-flow/) | Where does money go? What repeats every month? |
| **Investment account** | [investment-account/](examples/investment-account/) | What are my largest positions? How is the portfolio split? What do I invest on a schedule? |
| **Alerts** | [alerts/](examples/alerts/) | Should I take action? Am I above or below a recommended threshold? |

One insight usually combines **one analysis pattern** + **one domain**.

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
| **Time** | Up to trailing 24 calendar months, floored at earliest in-scope transaction (or single month / week / quarter) from transaction `date` |
| **Dimension** | `personal_finance_category_primary`, merchant, account, etc. |
| **Filters** | Exclude pending, removed, transfers/income/loan payments as needed; net refunds into category |
| **Output** | `months[]` + sparse `rows[]` + `month_totals[]` (or `period_totals[]`); chart series use `months[]` with metric fields |
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
| **Primary tables** | Holdings + securities |
| **Classification** | Rules on `plaid_investment_securities.type` or transaction category codes |
| **Output** | `asset_classes[]` or category shares; `total_*_value` |

**Examples**

| Insight | File |
|---|---|
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) *(share-of-month view)* |

**Variants you can build**

- Sector or geography allocation (needs enrichment beyond Plaid)
- Account-type mix (% in 401k vs brokerage vs IRA)
- Spend mix for a single month (same data as monthly-by-category)
- Liability mix (mortgage vs credit card vs student loan)

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

**Data shape:** Flat list of detected patterns grouped into product buckets (`bills`, `income`, `subscriptions`; `income` = `INCOME`; `bills` = `RENT_AND_UTILITIES` or `LOAN_PAYMENTS`; `subscriptions` = all other recurring outflows including uncategorized): merchant or security, frequency, typical amount, last occurrence. **Detection lookback** loads the last 24 months of transactions (required for annual and quarterly cadences); **6-month display window** (`window_start` / `window_end`) is metadata for the display window only.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_transactions` (spend/income) or `plaid_investment_transactions` (invest) |
| **Detection** | `detection_lookback_start` = today − 24 months; dual eligibility paths for outflow and inflow; group by merchant or account+security; ≥2 occurrences; median interval → frequency bucket; amount consistency (±20%); active filter from `last_date` |
| **Classification** | Assign `group`: `income` when `personal_finance_category_primary = INCOME`; `bills` when primary is `RENT_AND_UTILITIES` or `LOAN_PAYMENTS`; `subscriptions` for all remaining outflows including uncategorized |
| **Frequencies** | Weekly, biweekly, semi-monthly, monthly, quarterly, annual |
| **Output** | `recurrences[]`, `by_group[]`, `by_account[]`, `detection_lookback_start`, `window_start`, `window_end` |
| **Shared core** | [cash-flow-core.md](examples/cash-flow/cash-flow-core.md) — joined transaction table; callers apply scope, timeframe, and eligibility ([recurring transactions](examples/cash-flow/recurring-transactions.md), [late paycheck](examples/alerts/late-paycheck.md)) |

**Examples**

| Insight | File |
|---|---|
| Recurring transactions | [recurring-transactions.md](examples/cash-flow/recurring-transactions.md) |
| Recurring investments | [recurring-investments.md](examples/investment-account/recurring-investments.md) |

**Variants you can build**

- Late paycheck alert — [late-paycheck.md](examples/alerts/late-paycheck.md) (consume `group = income` from recurring transactions)
- Subscriptions only — filter `group = subscriptions`
- Subscription price increase alert — [subscription-price-increase.md](examples/alerts/subscription-price-increase.md)
- Estimated monthly recurring total (rollup row derived from `by_group[]`, `by_account[]`, or `recurrences[]`)
- Dormant recurrences (failed active filter — "used to pay X")

**Schema note:** Plaid has no recurring-transaction flag in the datatables; patterns are **inferred**. Document limitations in the spec (missed merges, variable bills, merchant name drift).

---

## Quick reference: all finalized examples

Grouped by domain. Full index: [examples/README.md](examples/README.md).

| Insight | Domain | Analysis pattern |
|---|---|---|
| Net worth snapshot | Net worth | Snapshot |
| Top 5 holdings | Investment | Snapshot + top N |
| Asset allocation | Investment | Snapshot + composition |
| Investment accounts by institution | Investment | Snapshot |
| Monthly spending by category | Cash flow | Period aggregation |
| Top 5 biggest purchases | Cash flow | Top N ranking |
| Recurring transactions | Cash flow | Recurring detection |
| Recurring investments | Investment | Recurring detection |
| Net worth balance chart | Net worth | Historical chart |
| Investment performance chart | Investment | Historical chart |
| Cash inflow and outflow chart | Cash flow | Period aggregation |
| Excess checking cash | Alerts | Snapshot |
| Subscription price increase | Alerts | Recurring detection + change |
| Late paycheck | Alerts | Recurring detection + lateness |

---

## Choosing a pattern for a new insight

1. **What is the user asking?** — "right now", "over time", "how much per month", "what repeats", "what's biggest", or "how is it split".
2. **What data exists?** — Balances need `plaid_accounts` history for charts; spend needs `plaid_transactions`; positions need holdings + securities.
3. **Snapshot or history?** — Latest `synced_at` vs point-in-time per day vs transaction `date` buckets.
4. **Detail + rollup rule** — Totals always derived from detail rows ([rollup convention](SKILL.md#rollup-convention) in SKILL.md).

When the insight is finalized, add a spec under [examples/](examples/README.md) and a row to the calibration table in [SKILL.md](SKILL.md).
