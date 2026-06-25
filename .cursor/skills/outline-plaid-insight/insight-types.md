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
| **Shared core** | [net-worth-core.md](examples/net-worth/net-worth-core.md) — `resolve_accounts_as_of` + `compute_net_worth`; account list via [GET /v1/account-balance](../../design-api/examples/net-worth/net-worth-apis.md#get-v1account-balance) (client sums by `group`) |

**Examples**

| Insight | File |
|---|---|
| Holdings by value | [holdings-by-value.md](examples/investment-account/holdings-by-value.md) |
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Investment accounts by institution | [investment-accounts-by-institution.md](examples/investment-account/investment-accounts-by-institution.md) |

**Variants you can build**

- Net worth account list — [GET /v1/account-balance](../../design-api/examples/net-worth/net-worth-apis.md#get-v1account-balance) returns flat `accounts[]`; client groups by `group` and sums `balance` for section subtotals
- Single-account balance — `GET /v1/account-balance?account_ids[]=<id>`
- Excess checking cash vs recommended minimum — [excess-checking-cash.md](examples/alerts/excess-checking-cash.md)
- Liabilities-only snapshot (credit cards + loans)
- Top N holdings (configurable cap on [holdings by value](examples/investment-account/holdings-by-value.md))
- Cash vs invested split across investment accounts

---

### 2. Historical balance chart (time series)

**Question:** How did a balance or total change over a selected period?

**Data shape:** Ordered `{ date, value }` points; parameterized `timeframe`.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_accounts` (retain all historical `synced_at` rows, not latest-only) |
| **Time** | Calendar day series; point-in-time account state per day (carry forward between syncs) |
| **Parameters** | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` — default `trailing_1y` |
| **Output** | `points[]`; series range from first/last point dates |
| **Shared core** | [net-worth-core.md](examples/net-worth/net-worth-core.md) — daily `resolve_accounts_as_of` + `compute_net_worth` per day; canonical route: [GET /v1/performance-history](../../design-api/examples/net-worth/net-worth-apis.md#get-v1performance-history) |

**Examples**

| Insight | File | Role |
|---|---|---|
| Net worth performance chart | [net-worth-apis.md](../../design-api/examples/net-worth/net-worth-apis.md) | `GET /v1/performance-history` — net worth mode |
| Investment performance chart | [investment-performance-chart.md](examples/investment-account/investment-performance-chart.md) | Client wrapper — all investment `account_ids` |

**Variants you can build**

- Total assets or total liabilities over time (same machinery, different roll-up)
- Single-account balance history — `account_ids: [id]` on performance history chart
- Multi-account subset — `account_ids: [...]` with signed rollup
- Spending or income cumulative line (transaction-derived, not balance snapshots)

---

### 3. Period aggregation (bucket + dimension)

**Question:** How much happened in each time bucket, broken down by a dimension (category, merchant, account)?

**Data shape:** Period picker metadata plus dimension breakdown for the selected period; period totals derived from dimension rows.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_transactions` |
| **Time** | Default trailing 12 calendar months (`trailing_1y`); up to 24 calendar months, floored at earliest in-scope transaction (or single month / week / quarter) from transaction `date` |
| **Dimension** | `personal_finance_category_primary`, merchant, account, etc. |
| **Filters** | Exclude pending, removed, transfers/income/loan payments as needed; net refunds into category |
| **Output** | `month` param + top-level `categories[]` + `months[]` as `string[]` (`YYYY-MM`) for picker insights; chart series use `months[]` as objects with metric fields (or `period_totals[]` for chart-only insights) |
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

**Data shape:** Classes with `value`, `percent`, and optional holdings drill-down; `total` = sum of parts; percents from total. `value` fields — 2 dp; `percent` fractions — 3 dp (see [SKILL.md output formatting](SKILL.md#output-formatting)).

| Aspect | Typical approach |
|---|---|
| **Primary tables** | Holdings + securities |
| **Classification** | Rules on `plaid_investment_securities.type` or transaction category codes |
| **Output** | `asset_classes[]` or category shares; `total_*_value` |

**Examples**

| Insight | File |
|---|---|
| Asset allocation | [asset-allocation.md](examples/investment-account/asset-allocation.md) |
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) *(share-of-month view via capped `categories[]` — top 5 + `OTHER` on selected month)* |

**Variants you can build**

- Assets vs liabilities stacked bar — [GET /v1/assets-liabilities](../../design-api/examples/net-worth/net-worth-apis.md#get-v1assets-liabilities)
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
| Top 5 biggest purchases | [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md) |
| Monthly spending by category | [monthly-spending-by-category.md](examples/cash-flow/monthly-spending-by-category.md) |

**Variants you can build**

- Top N holdings (cap sorted list from [holdings by value](examples/investment-account/holdings-by-value.md))
- Top individual purchases (trailing 30 days) — [top-5-biggest-purchases.md](examples/cash-flow/top-5-biggest-purchases.md)
- Top merchants by spend
- Top gainers/losers *(blocked or approximate without cost basis / historical prices)*
- Largest recurring bills (combine with recurring detection, then rank by `typical_amount`)

---

### 6. Recurring pattern detection

**Question:** What charges or contributions happen on a predictable schedule?

**Data shape:** Flat list of detected patterns grouped into product buckets (`bills`, `income`, `subscriptions`, `transfers`, `other_inflow`; `income` = inflow + `INCOME`; `transfers` = `TRANSFER_IN` or `TRANSFER_OUT`; `bills` = outflow + `RENT_AND_UTILITIES`, `LOAN_PAYMENTS`, `GENERAL_SERVICES`, `GOVERNMENT_AND_NON_PROFIT`, or `BANK_FEES`; `subscriptions` = all other recurring outflows including uncategorized; `other_inflow` = remaining inflows): merchant or security, frequency, typical amount, last occurrence, projected `next_date`, `occurrences[]` (actual last 3 calendar months + projected next 6 by default), and `account_mask`. Uses all available transaction history (no detection lookback cap). **Upcoming tab** — client filters `next_date >= today` or uses projected entries in `occurrences[]`.

| Aspect | Typical approach |
|---|---|
| **Primary tables** | `plaid_transactions` (spend/income) or `plaid_investment_transactions` (invest) |
| **Detection** | Full cash-flow-core table (no date lower bound, no upfront category/account/amount exclusions); set `direction` from amount sign; group by merchant or account+security; ≥2 occurrences; `median_gap_days` → frequency bucket; amount consistency (±20%); active filter from `last_date`; frequency-aware `next_date` projection; build `occurrences[]` timeline (actual + projected) |
| **Classification** | Assign `group` after detection from `direction` + category: `income`, `transfers`, `bills`, `subscriptions`, `other_inflow` (see [recurring transactions](examples/cash-flow/recurring-transactions.md) step 8) |
| **Frequencies** | Weekly, biweekly, monthly, annual |
| **Output** | `as_of`, `history_start`, `projection_end`, `recurrences[]` (`next_date`, `occurrences[]`, `account_mask`, `median_gap_days`, …), `by_group[]`, `by_account[]` |
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
| Holdings by value | Investment | Snapshot |
| Asset allocation | Investment | Snapshot + composition |
| Investment accounts by institution | Investment | Snapshot |
| Monthly spending by category | Cash flow | Period aggregation + Top N |
| Top 5 biggest purchases | Cash flow | Top N ranking |
| Recurring transactions | Cash flow | Recurring detection |
| Recurring investments | Investment | Recurring detection |
| Net worth performance chart | Net worth | Historical chart — [design-api](../../design-api/examples/net-worth/net-worth-apis.md) |
| Investment performance chart | Investment | Historical chart (wrapper) |
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
