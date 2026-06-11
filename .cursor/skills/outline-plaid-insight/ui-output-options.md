# UI output options

How insight **data output** maps to UI presentation patterns. Each insight data spec in [examples/](examples/README.md) may reference one or more UI options from this file.

**Convention:** Every UI node has a **label** (what it is) and a **balance** (the correlated number at that level). Parent balances are rollups of their children unless noted otherwise.

---

## UI output types

Insights are grouped below by UI output type. Pick the type that matches your data shape, then follow the insight-specific spec for columns, hierarchy, and build steps.

### Nested list

Best for snapshots, composition/allocation breakdowns, and top-N with drill-down. Renders a hierarchy where each node has a label and balance; parent balances roll up from children.

| Insight | Section |
|---|---|
| Net worth snapshot | [Net worth — nested list](#net-worth--nested-list) |
| Top 5 holdings | [Top 5 holdings — nested list](#top-5-holdings--nested-list) |
| Asset allocation | [Asset allocation — nested list](#asset-allocation--nested-list) |
| Investment accounts by institution | [Investment accounts by institution — nested list](#investment-accounts-by-institution--nested-list) |

### Flat table

Best for period aggregation, recurring pattern lists, and simple ranked tables without nesting. Single-level rows; column sets vary by insight.

| Insight | Section |
|---|---|
| Monthly spending by category | [Monthly spending by category — flat table](#monthly-spending-by-category--flat-table) |
| Recurring spending | [Recurring spending — flat table](#recurring-spending--flat-table) |
| Top 5 biggest purchases | [Top 5 biggest purchases — flat table](#top-5-biggest-purchases--flat-table) |
| Recurring investments | [Recurring investments — flat table](#recurring-investments--flat-table) |
| Cash account detail | [Account detail — flat table](#account-detail--flat-table) |

### Line chart

Best for historical balance or value time series. X-axis is date; Y-axis is currency; subtitle shows period change for the active timeframe.

| Insight | Section |
|---|---|
| Net worth balance chart | [Net worth balance chart — line chart](#net-worth-balance-chart--line-chart) |
| Investment performance chart | [Investment performance chart — line chart](#investment-performance-chart--line-chart) |

### Combo line and bar chart

Best for period aggregation with inflow/outflow bars and a net trend line overlaid on the same axis.

| Insight | Section |
|---|---|
| Cash inflow and outflow chart | [Cash inflow/outflow chart — combo line and bar chart](#cash-inflow-outflow-chart--combo-line-and-bar-chart) |

### Stacked bar

Best for snapshot composition bars where segments sum to 100% of a gross total. One horizontal bar with a legend below.

| Insight | Section |
|---|---|
| Assets / liabilities bar | [Assets / liabilities bar — stacked bar](#assets--liabilities-bar--stacked-bar) |

### Insight card

Best for snapshot alerts with a hero metric and optional account breakdown. Show only when a condition is met (e.g. `has_excess = true`).

| Insight | Section |
|---|---|
| Excess checking cash | [Excess checking cash — insight card](#excess-checking-cash--insight-card) |
| Subscription price increase | [Subscription price increase — insight card](#subscription-price-increase--insight-card) |
| Late paycheck | [Late paycheck — insight card](#late-paycheck--insight-card) |

### Composite

Best for full-screen layouts that combine multiple insight payloads (chart + list, chart + allocation bar + table).

| Insight | Section |
|---|---|
| Overview dashboard | [Overview dashboard — composite](#overview-dashboard--composite) |
| Investment account detail | [Investment account detail — composite](#investment-account-detail--composite) |

---

## Nested list

A hierarchical layout: parent nodes group children; each node shows a **label** and **balance** (or omits balance for section headers). Parent balances are rollups of their children unless noted otherwise. Use when the user needs to drill from a total into grouped detail.

### Net worth — nested list

A four-level breakdown: Assets and Liabilities as top-level section headers (no balance), category groups (Cash, Investment, Credit cards, Loans) with balances, individual accounts at the leaves. **No net worth total in this list** — show combined net worth from [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) (`points[last].net_worth`) in the same screen (e.g. chart header or hero above the list).

#### Hierarchy

```
Assets                               → (no balance — section header)
├── Cash                             → cash_balance
│   ├── {account name}               → account.balance
│   └── ...
└── Investment                       → investment_balance
    ├── {account name}               → account.balance
    └── ...
Liabilities                          → (no balance — section header)
├── Credit cards                     → credit_cards_balance
│   ├── {account name}               → account.balance
│   └── ...
└── Loans                            → loans_balance
    ├── {account name}               → account.balance
    └── ...
```

#### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Section | "Assets" | — (omit) | — |
| Group | "Cash" | `cash_balance` | `cash_balance`, `asset_groups[]` |
| Group | "Investment" | `investment_balance` | `investment_balance`, `asset_groups[]` |
| Leaf | `account.name` | `account.balance` | `asset_groups[].accounts[]`; optional subtitle `account.synced_at` |
| Section | "Liabilities" | — (omit) | — |
| Group | "Credit cards" | `credit_cards_balance` | `credit_cards_balance`, `liability_groups[]` |
| Group | "Loans" | `loans_balance` | `loans_balance`, `liability_groups[]` |
| Leaf | `account.name` | `account.balance` | `liability_groups[].accounts[]`; optional subtitle `account.synced_at` |

#### Build steps

1. Start from the [net worth snapshot data output](examples/net-worth/net-worth-snapshot.md) (`asset_groups`, `liability_groups`, category balances, `as_of`).
2. Render Assets and Liabilities as section headers (no balance column value) → category groups → account leaves.
3. Render asset category groups in order: Cash, Investment. Liability category groups: Credit cards, Loans. Skip groups with no accounts.
4. Verify category and account balances; each group `total_balance` = sum of child `balance`.
5. **Net worth total (sibling widget):** load [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) for the same user/timeframe; display `points[last].net_worth` (and optional `period_return_amount` / `period_return_pct` from the chart subtitle) above or beside this list — do not show `net_worth` from the snapshot payload as a list root row.
6. Optional footer on the list: "As of {as_of}" from snapshot `as_of` (align with chart `as_of` when both use the active window end).

#### Example (rendered)

*Net worth from net worth balance chart: **$122,500** (+3.6% trailing 6m)*

```
Assets
├── Cash                         $5,000
│   └── Chase Checking             $5,000
└── Investment                 $120,000
    └── Fidelity 401k              $120,000
Liabilities
└── Credit cards                   $2,500
    └── Chase Sapphire               $2,500
```

### Top 5 holdings — nested list

A two-level hierarchy: holding (security) at the top with total value, investment accounts nested below with each account's portion. Up to five holding groups, ordered by `total_value` descending.

#### Hierarchy

```
1. {security name} ({ticker})          → holding.total_value
   ├── {account name}                  → account.value
   └── ...
2. ...
```

#### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Holding (root per group) | `holding.name` (+ ticker) | `holding.total_value` | `holdings[]` |
| Account (child) | `account.account_name` | `account.value` | `holdings[].accounts[]` |

#### Build steps

1. Start from the [top 5 holdings data output](examples/investment-account/top-5-holdings.md) (`holdings`, `as_of`).
2. Render each holding in `rank` order (1–5).
3. Under each holding, render `accounts` in `value` descending order.
4. Verify `holding.total_value` = sum of `accounts[].value`.
5. Show `as_of` as subtitle or footnote.

#### Example (rendered)

```
1. Apple Inc. (AAPL)                 $48,200
   ├── Fidelity 401k                  $30,000
   └── Robinhood Brokerage            $18,200
2. Vanguard Total Stock Market (VTI)   $32,100
   └── Vanguard Brokerage             $32,100
3. Microsoft Corp. (MSFT)              $28,400
   └── Fidelity 401k                  $28,400
```

*As of May 26, 2026*

#### Notes

- Display rank prefix (1–5) on holding rows only, not on account children.
- Display ticker in parentheses when both `name` and `ticker_symbol` are present.
- Single-account holdings: show one child row (no flattening to a flat list).
- Fewer than 5 holdings: show only available holding groups.
- Tie-breaking when `total_value` is equal: sort holdings by `name` alphabetically.

### Asset allocation — nested list

A two-level hierarchy: portfolio allocation at the root, asset-class groups in the middle, individual securities at the leaves. Class balances are rollups of attributed holding values; the root shows `total_investment_value`. Unclassified positions appear as an **Other** class row with nested holdings — not as a footnote. Asset-class rows support toggling between **percent** and **dollar** display; holding leaves always show dollars.

#### Display mode (% / $)

| Mode | Class row shows | Root shows | Holding rows show |
|---|---|---|---|
| **$** (default) | `class.value` formatted as currency | `total_investment_value` | `holding.value` |
| **%** | `class.percent` formatted as percent (e.g. `percent × 100` → `42%`) | `total_investment_value` (unchanged) | `holding.value` (unchanged) |

- Provide a single control (segmented toggle, icon button, etc.) that switches all class rows between modes. Persist the user's choice for the session when possible.
- Show **one** metric per class row — do not show both percent and dollar on the same row.
- `percent` is a fraction (0–1) in data output; multiply by 100 for display. Round displayed percents to whole numbers unless the UI needs decimals for small slices.

#### Hierarchy

```
Portfolio allocation              → total_investment_value (always $)
├── {asset class label}           → class.value OR class.percent (per toggle)
│   ├── {security name} ({ticker}) → holding.value (always $)
│   └── ...
├── Other                         → other.value OR other.percent (when unclassified)
│   ├── {security name} ({ticker}) → holding.value
│   └── ...
└── ...
```

#### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Root | "Portfolio allocation" | `total_investment_value` | `total_investment_value` |
| Class (child) | Display label for `asset_class` (`other` → "Other") | `class.value` ($ mode) or `class.percent` (% mode) | `asset_classes[]` |
| Holding (leaf) | `holding.name` (+ ticker) | `holding.value` | `asset_classes[].holdings[]` |

#### Build steps

1. Start from the [asset allocation data output](examples/investment-account/asset-allocation.md) (`asset_classes`, `total_investment_value`, `as_of`).
2. Create root → class groups → holding leaves in `asset_classes` order (`value` descending).
3. Render each class row using the active display mode (`value` or `percent`), including **Other** when `asset_class = other`.
4. Under **Other**, render `holdings` in `value` descending order (same as other classes).
5. Verify `class.value` = sum of `holdings[].value` for that class; verify displayed percent matches `class.percent`.
6. Verify root `total_investment_value` = sum of all `asset_classes[].value`.
7. Show `as_of` as subtitle or footnote only — do not use a separate unclassified footnote.

#### Example (rendered)

**$ mode**

```
Portfolio allocation              $197,400
├── Equities                       $77,800
│   ├── Apple Inc. (AAPL)          $48,200
│   └── Vanguard Total Stock (VTI) $29,600
├── International equities         $33,400
│   └── Nestlé SA (NSRGY)          $33,400
├── Bonds                          $52,000
│   └── Vanguard Total Bond (BND)  $52,000
├── Cash                           $13,000
│   └── Money Market Fund          $13,000
├── Crypto                          $9,200
│   └── Bitcoin (BTC)               $9,200
└── Other                          $12,000
    └── Target Date 2050 (FDFAX)   $12,000
```

**% mode**

```
Portfolio allocation              $197,400
├── Equities                         39%
│   ├── Apple Inc. (AAPL)          $48,200
│   └── Vanguard Total Stock (VTI) $29,600
├── International equities           17%
│   └── Nestlé SA (NSRGY)          $33,400
├── Bonds                            26%
│   └── Vanguard Total Bond (BND)  $52,000
├── Cash                              7%
│   └── Money Market Fund          $13,000
├── Crypto                            5%
│   └── Bitcoin (BTC)               $9,200
└── Other                             6%
    └── Target Date 2050 (FDFAX)   $12,000
```

*As of May 26, 2026.*

#### Notes

- Map `asset_class` codes to display labels in the UI layer (e.g. `international_equity` → "International equities", `other` → "Other").
- **Other** holds securities missing fund-composition enrichment; omit the class row when `unclassified_value` is 0.
- Display ticker in parentheses when both `name` and `ticker_symbol` are present.
- A security may appear under multiple classes when enrichment splits an ETF across classes.
- Omit asset classes with zero `value`.
- Tie-breaking when `value` is equal: sort classes and holdings by display label alphabetically.
- Class percents are shares of `total_investment_value`; all class rows (including **Other**) sum to 100% in % mode.

### Investment accounts by institution — nested list

A two-level hierarchy: institutions at the top, individual investment accounts nested below. Institution balances are rollups of child account balances. **No portfolio total in this list** — show combined value from [investment performance chart](examples/investment-account/investment-performance-chart.md) (`end_value`) in the same screen (e.g. chart header or hero above the list).

#### Hierarchy

```
{institution_name}                       → institution.total_balance
├── {account name}                       → account.balance
└── ...
```

#### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Group | `institution.institution_name` | `institution.total_balance` | `institutions[]` |
| Leaf | `account.name` (+ optional ` ••{mask}`) | `account.balance` | `institutions[].accounts[]` |

#### Build steps

1. Start from the [investment accounts by institution data output](examples/investment-account/investment-accounts-by-institution.md) (`institutions`, `as_of`).
2. Render institution groups in `total_balance` descending order; nest `accounts` under each.
3. Within each institution, render `accounts` in `balance` descending order.
4. Verify `institution.total_balance` = sum of `accounts[].balance`.
5. **Portfolio total (sibling widget):** load [investment performance chart](examples/investment-account/investment-performance-chart.md) for the same user/timeframe; display `end_value` (and optional `period_return_amount` / `period_return_pct`) above or beside this list — do not sum institution rows for the headline total.
6. Optional footer on the list: "As of {as_of}" from `as_of` (align with chart `as_of` when both use latest snapshot).

#### Example (rendered)

*Portfolio total from investment performance chart: **$186,600** (+3.5% trailing 6m)*

```
Fidelity                     $120,000
├── 401k Plan                 $95,000
└── Brokerage ••4821          $25,000
Vanguard                      $66,600
├── Traditional IRA           $48,200
└── Roth IRA                  $18,400
```

#### Notes

- Append ` ••{mask}` to the account label when `mask` is present.
- Show `subtype` as a secondary label or badge (e.g. "401k", "ira") when useful for disambiguation.
- `balance_source = holdings` accounts: optional subtle indicator that balance was derived from positions (institution did not report account-level balance).
- Single account under one institution: one parent row with one child (no flattening to a flat list).
- Empty `institutions`: show an empty state (no linked investment accounts).
- Drill-down to holdings: use [top 5 holdings](examples/investment-account/top-5-holdings.md) or [asset allocation](examples/investment-account/asset-allocation.md).

---

## Flat table

A single-level table: one row per item, no nesting. Column sets and sort order vary by insight. Use for ranked lists, recurring patterns, and period breakdowns where drill-down is not needed.

### Monthly spending by category — flat table

A single-level table for the **current calendar month** only: one row per category. No nesting, no per-month total row, and no `Month` column (month is in the subtitle).

#### Columns

| Column | Source |
|---|---|
| Category | `row.category` (`personal_finance_category_primary`) |
| Amount | `row.amount` (net spend for that category in the current month) |

#### Build steps

1. Start from the [monthly spending data output](examples/cash-flow/monthly-spending-by-category.md) (`month`, `rows`, `as_of`).
2. Render `rows` in order (highest `amount` first). All rows are for `month`.
3. Subtitle: current month label from `month` or `as_of` (e.g. "May 2026 · as of May 1, 2026").

#### Example (rendered)

*May 2026 · as of May 1, 2026*

| Category | Amount |
|---|---|
| FOOD_AND_DRINK | $412.00 |
| TRANSPORTATION | $186.50 |
| GENERAL_MERCHANDISE | $95.20 |

#### Notes

- Data is scoped to the current calendar month only — do not show prior months in this table.
- Categories are ordered by amount descending (highest spend at top).
- Category labels: map Plaid codes (e.g. `FOOD_AND_DRINK`) to display names in the UI layer.
- Negative net category amounts (refunds exceeded spend) display as negative values in the Amount column.
- `as_of` is the first calendar day of the current month (`YYYY-MM-01`), not the latest transaction date.

### Recurring spending — flat table

One row per active recurring spend pattern, sorted by frequency then amount.

#### Columns

| Column | Source |
|---|---|
| Merchant | `recurrence.merchant_name` |
| Account | `recurrence.account_name` |
| Category | `recurrence.category` (optional column) |
| Frequency | `recurrence.frequency` (`weekly`, `biweekly`, `semi-monthly`, `monthly`, `quarterly`, `annual`) |
| Amount | `recurrence.typical_amount` |
| Last date | `recurrence.last_date` |

#### Build steps

1. Start from the [recurring spending data output](examples/cash-flow/recurring-spending.md) (`recurrences`, `window_start`, `window_end`).
2. Render each item in `recurrences` as one table row (pre-sorted).
3. Optional footer: "Patterns detected from {window_start} to {window_end}".

#### Example (rendered)

| Merchant | Account | Category | Frequency | Amount | Last date |
|---|---|---|---|---|---|
| Netflix | Chase Sapphire | ENTERTAINMENT | Monthly | $15.99 | 2026-05-01 |
| Spotify | Chase Checking | ENTERTAINMENT | Monthly | $11.99 | 2026-05-03 |
| PG&E | Chase Checking | RENT_AND_UTILITIES | Monthly | $142.50 | 2026-04-28 |

*Patterns detected from Nov 26, 2025 to May 26, 2026*

#### Notes

- Map `frequency` to display labels (e.g. `semi-monthly` → "Semi-monthly").
- Optional column **Occurrences** from `occurrence_count` for power users.
- **Category** can be hidden in compact layouts.

### Top 5 biggest purchases — flat table

A single-level table for the **trailing 30 days**: one row per ranked purchase (up to 5). Rows are individual transactions, not merchant rollups.

#### Columns

| Column | Source |
|---|---|
| Merchant | `purchase.merchant_name` |
| Category | `purchase.category` |
| Amount | `purchase.amount` |
| Date | `purchase.date` |
| Account | `purchase.account_name` (optional column) |

#### Build steps

1. Start from the [top 5 biggest purchases data output](examples/cash-flow/top-5-biggest-purchases.md) (`purchases`, `window_start`, `window_end`).
2. Render each item in `purchases` as one table row in `rank` order (1–5).
3. Subtitle: `{window_start} to {window_end}` (e.g. "Apr 28, 2026 to May 28, 2026").
4. Optional footer: total spend from `period_total_spend` when present.

#### Example (rendered)

*Apr 28, 2026 to May 28, 2026*

| Merchant | Category | Amount | Date | Account |
|---|---|---|---|---|
| Apple Store | GENERAL_MERCHANDISE | $1,299.00 | 2026-05-12 | Chase Sapphire |
| Delta Air Lines | TRAVEL | $842.50 | 2026-05-03 | Chase Sapphire |
| Best Buy | GENERAL_MERCHANDISE | $649.99 | 2026-04-30 | Wells Fargo Checking |
| Whole Foods | FOOD_AND_DRINK | $412.18 | 2026-05-20 | Amex Gold |
| IKEA | HOME_IMPROVEMENT | $387.44 | 2026-05-08 | Wells Fargo Checking |

#### Notes

- Show rank prefix (1–5) on rows or as a leading column when space allows.
- Fewer than 5 eligible purchases: show only available rows.
- Map Plaid category codes (e.g. `FOOD_AND_DRINK`) to display names in the UI layer.
- **Account** can be hidden in compact layouts.
- Tie-breaking when `amount` is equal: most recent `date` first (matches data sort).

### Recurring investments — flat table

One row per active recurring investment pattern, sorted by frequency then amount.

#### Columns

| Column | Source |
|---|---|
| Account | `recurrence.account_name` |
| Security | `recurrence.security_name` (+ `ticker_symbol` if present) |
| Frequency | `recurrence.frequency` (`weekly`, `biweekly`, `semi-monthly`, `monthly`, `quarterly`, `annual`) |
| Amount | `recurrence.typical_amount` |
| Last date | `recurrence.last_date` |

#### Build steps

1. Start from the [recurring investments data output](examples/investment-account/recurring-investments.md) (`recurrences`, `window_start`, `window_end`).
2. Render each item in `recurrences` as one table row (pre-sorted).
3. Optional footer: "Patterns from {window_start} to {window_end}".

#### Example (rendered)

| Account | Security | Frequency | Amount | Last date |
|---|---|---|---|---|
| Fidelity 401k | Vanguard Target 2055 (VTTVX) | Monthly | $500.00 | 2026-05-01 |
| Robinhood | Apple Inc. (AAPL) | Biweekly | $100.00 | 2026-05-20 |
| Fidelity 401k | Vanguard Total Stock (VTI) | Semi-monthly | $250.00 | 2026-05-15 |

*Patterns detected from Nov 26, 2025 to May 26, 2026*

#### Notes

- Map `frequency` to display labels (e.g. `semi-monthly` → "Semi-monthly").
- Optional column **Occurrences** from `occurrence_count` for power users.

---

## Line chart

A time-series line chart: one point per calendar day. X-axis is date; Y-axis is currency. Header or subtitle shows period dollar and percent change for the active timeframe. Y-axis scale is derived by the chart library from the series points (investment performance chart may still expose `total_value_min` / `total_value_max` in its data output).

### Net worth balance chart — line chart

A time-series line chart: one point per calendar day showing total net worth. The X-axis is date; the Y-axis is net worth balance. A subtitle shows period dollar and percent change for the active timeframe.

#### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Net worth" |
| Hero value | `points[last].net_worth` | Currency (e.g. `$122,500`) — latest net worth at `as_of` |
| Subtitle — dollar change | `period_return_amount` | Signed currency (e.g. `+$4,300`) |
| Subtitle — percent change | `period_return_pct` | Signed percent (`period_return_pct × 100`, e.g. `+3.6%`) |
| Subtitle — window | `timeframe`, `as_of` | Human label (e.g. "Trailing 6 months · as of May 26, 2026") |

Example header: **$122,500** · `+$4,300 (+3.6%) · Trailing 6 months · as of May 26, 2026`

Omit percent segment when `period_return_pct` is absent (`points[0].net_worth` ≤ 0).

#### Axes

| Axis | Source | Format |
|---|---|---|
| X | `point.date` | Date (e.g. May 26 or 05/26) |
| Y | `point.net_worth` | Currency |

Derive Y-axis domain from `MIN(points[].net_worth)` and `MAX(points[].net_worth)` at render time. Do not pad to zero or use a fixed global scale.

#### Build steps

1. Start from the [net worth balance chart data output](examples/net-worth/net-worth-balance-chart.md) (`points`, `period_return_amount`, `period_return_pct`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Render title and subtitle from header table above.
3. Map each item in `points` to a chart coordinate: `(date, net_worth)`.
4. Set chart Y-axis domain from `MIN` / `MAX` of `points[].net_worth`.
5. Connect points in array order (ascending `date`).
6. Format Y-axis labels as currency; choose X-axis tick density based on `timeframe` (e.g. weekly ticks for `trailing_1m`, monthly for `trailing_1y`).
7. Optional **timeframe** control (`trailing_1m`, `trailing_3m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time`): re-run the insight with the selected `timeframe` and re-render (updates `points` and return fields).

#### Example (rendered)

| Date | Net worth |
|---|---|
| 2025-11-26 | $118,200 |
| 2025-12-01 | $119,400 |
| 2026-01-15 | $120,800 |
| 2026-03-01 | $121,500 |
| 2026-05-01 | $122,100 |
| 2026-05-26 | $122,500 |

*+$4,300 (+3.6%) · Trailing 6 months · as of May 26, 2026*

#### Notes

- Y-axis range is scoped to the active timeframe — when `timeframe` changes, recompute domain from the new `points[]`.
- When all values are equal, use that single value for both bounds (flat line); add minimal visual padding only if the chart library requires it.
- `points` must be sorted ascending by `date`; do not re-sort in the UI layer.
- Single point: render as a dot or short flat segment; period return is zero.
- Period return is simple holding-period change on net worth balances, not time-weighted return.
- Empty `points`: show an empty state (no linked accounts or no history in range).
- Between syncs, flat segments are expected (carry-forward balances).
- Drill-down to per-account balances: use [net worth snapshot](examples/net-worth/net-worth-snapshot.md), not this chart payload.
- **Net worth headline:** On net worth overview screens, show `points[last].net_worth` (and period return from the subtitle) as the primary total; pair with [net worth snapshot](examples/net-worth/net-worth-snapshot.md) for the account breakdown (no net worth root row on the list).

### Investment performance chart — line chart

A time-series line chart: one point per calendar day showing combined investment account value. The X-axis is date; the Y-axis is total value. A subtitle shows period dollar and percent return for the active timeframe.

#### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Investment performance" |
| Subtitle — dollar change | `period_return_amount` | Signed currency (e.g. `+$4,200`) |
| Subtitle — percent change | `period_return_pct` | Signed percent (`period_return_pct × 100`, e.g. `+3.5%`) |
| Subtitle — window | `timeframe`, `window_end` | Human label (e.g. "Trailing 6 months · as of May 26, 2026") |

Example subtitle: `+$4,200 (+3.5%) · Trailing 6 months · as of May 26, 2026`

Omit percent segment when `period_return_pct` is absent (`start_value` ≤ 0).

#### Axes

| Axis | Source | Format |
|---|---|---|
| X | `point.date` | Date (e.g. May 26 or 05/26) |
| Y | `point.total_value` | Currency |
| Y domain (min) | `total_value_min` | Lower bound of Y-axis for this timeframe |
| Y domain (max) | `total_value_max` | Upper bound of Y-axis for this timeframe |

Set the Y-axis range to `[total_value_min, total_value_max]` — the min and max `total_value` values in `points` for the selected timeframe. Do not pad to zero or use a fixed global scale.

#### Build steps

1. Start from the [investment performance chart data output](examples/investment-account/investment-performance-chart.md) (`points`, `total_value_min`, `total_value_max`, `period_return_amount`, `period_return_pct`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Render title and subtitle from header table above.
3. Map each item in `points` to a chart coordinate: `(date, total_value)`.
4. Set chart Y-axis domain to `total_value_min` (bottom) and `total_value_max` (top). Verify they match `MIN(points[].total_value)` and `MAX(points[].total_value)`.
5. Connect points in array order (ascending `date`).
6. Format Y-axis labels as currency; choose X-axis tick density based on `timeframe` (e.g. weekly ticks for `trailing_1m`, monthly for `trailing_1y`).
7. Optional **timeframe** control (`trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time`): re-run the insight with the selected `timeframe` and re-render (updates `points`, bounds, and return fields).

#### Example (rendered)

| Date | Total value |
|---|---|
| 2025-11-26 | $82,400 |
| 2025-12-01 | $83,100 |
| 2026-01-15 | $84,500 |
| 2026-03-01 | $85,200 |
| 2026-05-01 | $86,100 |
| 2026-05-26 | $86,600 |

*+$4,200 (+5.1%) · Trailing 6 months · as of May 26, 2026*

#### Notes

- Y-axis range is always scoped to the active timeframe — when `timeframe` changes, recompute domain from the new `total_value_min` / `total_value_max`.
- When all values are equal (`total_value_min` = `total_value_max`), use that single value for both bounds (flat line); add minimal visual padding only if the chart library requires it.
- `points` must be sorted ascending by `date`; do not re-sort in the UI layer.
- Single point: render as a dot or short flat segment; period return is zero.
- Empty `points`: show an empty state (no linked investment accounts or no history in range).
- Between syncs, flat segments are expected (carry-forward balances).
- Period return is simple holding-period return on balances, not time-weighted return.
- Drill-down to holdings composition: use [top 5 holdings](examples/investment-account/top-5-holdings.md) or [asset allocation](examples/investment-account/asset-allocation.md), not this chart payload.
- **Portfolio headline:** On investment overview screens, show `end_value` (and period return from the subtitle) as the primary total; pair with [investment accounts by institution](examples/investment-account/investment-accounts-by-institution.md) for the account breakdown (no duplicate root total on the list).

---

## Combo line and bar chart

A layered combo chart: grouped bars per time bucket (e.g. calendar month) for positive and negative series, with a line overlay for a net or trend series on the same axis. Use when the user needs to compare inflow vs outflow while seeing net change over time.

### Cash inflow/outflow chart — combo line and bar chart

A layered combo chart: grouped bars per calendar month for cash inflow (positive) and cash outflow (negative), with a line overlay for net cash flow. The data payload spans the trailing 24 months; the chart renders a **6-month sliding viewport** with arrow navigation. The X-axis shows six visible months; the Y-axis is currency.

#### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Cash flow" |
| Hero / primary net | `months[start_index + 5].net_cash_flow` | Signed currency for **rightmost visible month** |
| Subtitle — date range | `months[start_index].month` … `months[start_index + 5].month` | e.g. `Dec 2025 – May 2026` |
| Subtitle — as of | `as_of` | `as of May 26, 2026` |

Example header (default viewport, May rightmost): **`+$1,600`** · `Dec 2025 – May 2026 · as of May 26, 2026`

Do not bind header to `timeframe`. Omit header when `months` is empty.

#### Navigation

| Control | Action |
|---|---|
| **Left arrow** (older) | `start_index -= 1`, clamp to `0` |
| **Right arrow** (newer) | `start_index += 1`, clamp to `months.length − 6` |
| **Initial `start_index`** | `max(0, months.length − 6)` — defaults to most recent 6 months |
| **Disabled state** | Left disabled at `start_index = 0`; right disabled at `start_index = months.length − 6` |
| **When `months.length ≤ 6`** | Show all months; hide or disable both arrows |

Viewport slice: `visible_months = months[start_index : start_index + 6]` (always 6 entries when `months.length ≥ 6`).

`start_index` is UI state only — not a data output field. Arrow navigation is client-side; do not re-fetch on scroll.

#### Axes

| Axis | Source | Format |
|---|---|---|
| X | `visible_months[].month` | Month label (e.g. `Dec 2025` or `12/25`); one tick per visible month (6 ticks) |
| Y — bars | `cash_inflow`, `cash_outflow` | Currency; inflow above zero, outflow below zero |
| Y — line | `net_cash_flow` | Currency; same scale as bars |
| Y domain (min) | `value_min` | Lower bound — min of `cash_outflow` and `net_cash_flow` across **all** `months` |
| Y domain (max) | `value_max` | Upper bound — max of `cash_inflow` and `net_cash_flow` across **all** `months` |

Set the Y-axis range to `[value_min, value_max]` from the full payload (stable while scrolling). Include zero on the axis when it falls within the domain. Do not pad to a fixed global scale.

#### Series

| Series | Type | Source | Visual |
|---|---|---|---|
| Inflow | Bar | `cash_inflow` | Positive bar above zero (e.g. green) |
| Outflow | Bar | `cash_outflow` | Negative bar below zero (e.g. red) |
| Net cash flow | Line | `net_cash_flow` | Line/markers overlaid on bars, distinct color |

Group inflow and outflow bars by `month` on the X-axis (side-by-side or overlapping per library). Draw the net line on top of the bar layer. Render series from `visible_months` only.

#### Build steps

1. Start from the [cash inflow and outflow chart data output](examples/cash-flow/cash-inflow-outflow-chart.md) (`months`, `value_min`, `value_max`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Set `start_index = max(0, months.length − 6)`.
3. Slice `visible_months = months[start_index : start_index + 6]`.
4. Render title and header from header table above (hero net from `months[start_index + 5]`, date range from first and last visible month).
5. For each item in `visible_months`, render bars at `(month, cash_inflow)` and `(month, cash_outflow)`.
6. Map each item in `visible_months` to a line coordinate: `(month, net_cash_flow)`; connect in array order (ascending `month`).
7. Set chart Y-axis domain to `value_min` (bottom) and `value_max` (top) from the full payload. Verify they match `MIN(months[].cash_outflow, months[].net_cash_flow)` and `MAX(months[].cash_inflow, months[].net_cash_flow)`.
8. Add a horizontal zero reference line when zero is within `[value_min, value_max]`.
9. Format Y-axis labels as currency; use one tick per visible month on the X-axis (6 ticks).
10. Wire left/right arrows to shift `start_index` by ±1 (clamped), re-slice `visible_months`, and re-render chart and header.
11. Legend: **Inflow**, **Outflow**, **Net cash flow**.

#### Example (rendered)

*Payload: 24 months in `months[]` (Jun 2024 – May 2026). Default viewport shows the last 6 rows below.*

| Month | Inflow | Outflow | Net |
|---|---|---|---|
| 2025-12 | $8,200 | −$6,400 | +$1,800 |
| 2026-01 | $8,100 | −$7,200 | +$900 |
| 2026-02 | $8,000 | −$8,500 | −$500 |
| 2026-03 | $8,300 | −$6,900 | +$1,400 |
| 2026-04 | $8,150 | −$7,100 | +$1,050 |
| 2026-05 | $8,400 | −$6,800 | +$1,600 |

*Default header: **+$1,600** · Dec 2025 – May 2026 · as of May 26, 2026*

*After one left-arrow click (`start_index` decremented): visible months Nov 2025 – Apr 2026; header hero net = Apr `net_cash_flow` (**+$1,050**) · Nov 2025 – Apr 2026 · as of May 26, 2026*

#### Notes

- Y-axis range is scoped to the full `months` payload — recompute `value_min` / `value_max` when data refreshes, not when the viewport scrolls.
- Header net follows the **right edge** of the viewport (rightmost visible month's `net_cash_flow`).
- When all values are equal (`value_min` = `value_max`), use that single value for both bounds (flat chart); add minimal visual padding only if the chart library requires it.
- `months` must be sorted ascending by `month`; do not re-sort in the UI layer.
- Empty `months`: show an empty state (no linked accounts or no transactions in range).
- Months with no transactions still appear with zero bars and zero net (per data spec).
- Zero-activity months still occupy bar slots within the 24-month window.
- Drill-down to category spend: use [monthly spending by category](examples/cash-flow/monthly-spending-by-category.md), not this chart payload.

---

## Stacked bar

A single horizontal bar divided into segments whose widths are proportional to each segment's share of a gross total. A legend below shows label and currency value per segment. Use for snapshot composition views where segments always sum to 100% of the bar.

### Assets / liabilities bar — stacked bar

A horizontal stacked bar comparing total assets and total liabilities as shares of gross exposure (`total_assets + total_liabilities`). Legend shows dollar values per segment. Pair with [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) for the net worth headline — segment widths are not sized by net worth.

#### Bar and legend

| Element | Source | Format |
|---|---|---|
| Bar — Assets segment | `segments` where `segment = assets` | Width = `percent × 100%` of bar |
| Bar — Liabilities segment | `segments` where `segment = liabilities` | Width = `percent × 100%` of bar |
| Legend — Assets | `segments[].value` where `segment = assets` | Color dot + "Assets" + currency (e.g. `$125,000.00`) |
| Legend — Liabilities | `segments[].value` where `segment = liabilities` | Color dot + "Liabilities" + currency |
| Net worth headline (sibling) | `net_worth` from chart or this payload | Currency — do not render on the bar itself |

#### Build steps

1. Start from the [assets / liabilities bar data output](examples/net-worth/assets-liabilities-bar.md) (`segments`, `total_assets`, `total_liabilities`, `net_worth`, `bar_total`, `as_of`).
2. Render one horizontal bar; left segment = Assets, right = Liabilities; rounded outer corners; thin gap between segments when both present.
3. Render legend below: color dot + label + currency value per segment in `segments` order (`assets`, then `liabilities`).
4. When only one segment, render full-width bar (no zero-width sliver for the omitted segment).
5. When `segments` is empty (`bar_total = 0`), show an empty state.
6. Optional footer: "As of {as_of}".

#### Example (rendered)

```
[████████████ Assets ████████|████ Liabilities ████]
  ● Assets          ● Liabilities
    $125,000.00       $50,000.00
```

*Net worth headline (sibling): **$75,000***

#### Notes

- Segment widths use `bar_total` (`total_assets + total_liabilities`), not `net_worth`.
- `percent` is a fraction (0–1) in data output; multiply by 100 for optional percent display.
- When paired with [net worth snapshot](examples/net-worth/net-worth-snapshot.md) on the overview dashboard, `total_assets` and `total_liabilities` must match snapshot totals at the same `as_of`.
- Do not recompute segment values or percents in the UI layer — use `segments[]` from the data payload.

---

## Insight card

A compact alert card with a hero metric, supporting context, and optional account breakdown. Render only when a trigger condition is met (e.g. `has_excess = true`). Use for actionable snapshot insights that do not need a full list or chart.

### Excess checking cash — insight card

Surfaces when combined checking balance exceeds a recommended cushion (115% of trailing 3-month average monthly spend). Hero shows `excess_cash`; body explains `recommended_minimum` and optional per-account breakdown.

#### Visibility

| Condition | Action |
|---|---|
| `has_excess = true` | Show card |
| `has_excess = false` | Hide card (no empty state) |
| No checking accounts | Hide card |
| `total_spend_3mo = 0` | Hide card or show neutral "insufficient spend history" state |

#### Card layout

| Element | Source | Format |
|---|---|---|
| Title | Static | "Excess checking cash" |
| Hero value | `excess_cash` | Currency (e.g. `$2,400 above recommended`) |
| Body | Derived | "Your checking balance is {total_checking_balance} — {recommended_minimum} recommended (115% of {monthly_avg_spend}/mo avg spend)" |
| Account list (optional) | `checking_accounts[]` | `{name}` + `balance`, sorted by `balance` descending |
| Footer | `as_of` | "As of {as_of}" |

#### Build steps

1. Start from the [excess checking cash data output](examples/alerts/excess-checking-cash.md) (`has_excess`, `excess_cash`, `total_checking_balance`, `recommended_minimum`, `monthly_avg_spend`, `checking_accounts`, `as_of`).
2. If `has_excess` is false, do not render.
3. Display `excess_cash` as the primary hero metric.
4. Optional expandable section: list `checking_accounts` with name (and ` ••{mask}` when present) and balance.
5. Footer from `as_of`.

#### Example (rendered)

**$2,400** above recommended

Your checking balance is **$8,400** — **$6,000** recommended (115% of $1,739/mo avg spend)

```
Chase Checking ••1234          $5,200
Wells Fargo Checking ••5678    $3,200
```

*As of May 26, 2026*

#### Notes

- Savings accounts are excluded from the balance comparison by design.
- Do not show the card when spend history is insufficient unless product chooses the neutral state.
- `recommended_minimum` and `excess_cash` are derived from detail rows — do not recompute in the UI layer.

### Subscription price increase — insight card

Surfaces when one or more active subscription-like recurring charges increased by **≥ 5%** vs the median of prior charges. Hero shows `estimated_monthly_impact`; body lists each affected merchant with before/after amounts.

#### Visibility

| Condition | Action |
|---|---|
| `has_increases = true` | Show card |
| `has_increases = false` | Hide card (no empty state) |
| No recurring subscription patterns in window | Hide card |

#### Card layout

| Element | Source | Format |
|---|---|---|
| Title | Static | "Subscription price increases" |
| Hero value | `estimated_monthly_impact` | Currency (e.g. `+$4.00/mo`) |
| Subtitle | `increase_count` | "{increase_count} subscription(s) increased" |
| Increase list | `increases[]` | `{merchant_name}` — `{prior_median} → {latest_amount}` (+{increase_pct}%), last {latest_date} |
| Footer | `window_end` | "Based on charges through {window_end}" |

#### Build steps

1. Start from the [subscription price increase data output](examples/alerts/subscription-price-increase.md) (`has_increases`, `increase_count`, `estimated_monthly_impact`, `increases`, `window_end`).
2. If `has_increases` is false, do not render.
3. Display `estimated_monthly_impact` as the primary hero metric (prefix with `+`, suffix with `/mo`).
4. Show subtitle from `increase_count`.
5. List `increases[]` sorted by `monthly_equivalent_increase` descending (already sorted in data output).
6. Footer from `window_end`.

#### Example (rendered)

**+$4.00/mo** from price increases

2 subscriptions increased

```
Netflix          $15.99 → $17.99  (+12.5%)   May 1
Spotify          $11.99 → $12.99  (+8.3%)    May 3
```

*Based on charges through May 28, 2026*

#### Notes

- Subscriptions are inferred from transaction history — not a Plaid recurring flag.
- `estimated_monthly_impact` is derived from `increases[]` — do not recompute in the UI layer.
- Annual and quarterly subscriptions are normalized to monthly equivalents in the data layer.

### Late paycheck — insight card

Surfaces when a detected recurring paycheck is **≥ 3 calendar days past** its predicted next pay date with no matching deposit. Hero shows `days_late` for the most overdue source; additional late sources listed when `late_count > 1`.

#### Visibility

| Condition | Action |
|---|---|
| `has_late_paychecks = true` | Show card |
| `has_late_paychecks = false` | Hide card (no empty state) |
| No active paycheck patterns | Hide card |

#### Card layout

| Element | Source | Format |
|---|---|---|
| Title | Static | "Late paycheck" when `late_count = 1`; "Late paychecks" when `late_count > 1` |
| Hero value | `late_paychecks[0].days_late` | "{days_late} days late" |
| Subtitle | `late_paychecks[0].payer_name` | "Expected from {payer_name}" |
| Body | `late_paychecks[0]` | "Typical amount {typical_amount}, last received {last_date}; expected {expected_next_date}" |
| Additional sources | `late_paychecks[1..]` | Same subtitle/body format per row when `late_count > 1` |
| Footer | `window_end` | "Based on deposits through {window_end}" |

#### Build steps

1. Start from the [late paycheck data output](examples/alerts/late-paycheck.md) (`has_late_paychecks`, `late_count`, `late_paychecks`, `window_end`).
2. If `has_late_paychecks` is false, do not render.
3. Use `late_paychecks[0]` (most overdue — already sorted by `days_late` descending) for hero, subtitle, and primary body.
4. When `late_count > 1`, list remaining entries from `late_paychecks[1..]`.
5. Footer from `window_end`.

#### Example (rendered)

**3 days late**

Expected from **Acme Corp Payroll**

Typical amount **$2,450**, last received **May 1**; expected **May 15**

*Based on deposits through May 28, 2026*

#### Example (multiple sources)

**Late paychecks**

**5 days late** — Expected from **Acme Corp Payroll**  
Typical amount **$2,450**, last received **May 1**; expected **May 15**

**3 days late** — Expected from **Side Gig LLC**  
Typical amount **$800**, last received **May 10**; expected **May 20**

*Based on deposits through May 28, 2026*

#### Notes

- Paycheck cadence is inferred from transaction history — not a Plaid recurring flag.
- `has_late_paychecks` and `late_count` are derived from `late_paychecks[]` — do not recompute in the UI layer.
- Each employer/payer is a separate row in `late_paychecks[]`; the card surfaces all late sources in one card.

---

## Composite

Full-screen layouts built from multiple insight payloads. Each subsection maps to a region of the screen.

### Overview dashboard — composite

Pairs [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) (hero + line chart + period return) with [net worth snapshot](examples/net-worth/net-worth-snapshot.md) (nested account list) and an assets/liabilities ratio bar. See [overview dashboard](examples/net-worth/overview-dashboard.md).

#### Regions

| Region | Source | Key fields |
|---|---|---|
| Net worth hero | `chart.points[last].net_worth` | Latest net worth |
| YTD / period change | `chart.period_return_amount`, `period_return_pct` | Active or YTD timeframe |
| Line chart | `chart.points[]` | `{ date, net_worth }` |
| Assets / liabilities bar | `bar.segments[]` from [assets / liabilities bar](examples/net-worth/assets-liabilities-bar.md) | `{ segment, label, value, percent }` |
| Account list | `snapshot.asset_groups`, `snapshot.liability_groups` + enriched `accounts[]` | Institution, mask, balance, `synced_at` |

#### Build steps

1. Load [overview dashboard](examples/net-worth/overview-dashboard.md) payload (or run chart + snapshot + enrichment separately).
2. Render chart header from `chart` using [net worth balance chart — line chart](#net-worth-balance-chart--line-chart) header rules.
3. Render account list from [net worth — nested list](#net-worth--nested-list); append institution and "Updated X ago" on leaf rows from enriched `accounts[]`.
4. Render assets/liabilities bar using [Assets / liabilities bar — stacked bar](#assets--liabilities-bar--stacked-bar) from `bar.segments[]` (or derive from snapshot `total_assets` / `total_liabilities` via the bar insight calculation).

### Account detail — flat table

Single-account header block plus a flat transaction or activity list. See [cash account detail](examples/net-worth/cash-account-detail.md).

| Column | Source |
|---|---|
| Title | `title` |
| Institution | `institution_name` |
| Updated | Relative time from `synced_at` |
| Available balance | `balances_available` (hide row if null) |
| Account type | Formatted `subtype` |
| Transaction name | `transactions[].display_name` |
| Date | `transactions[].date` |
| Amount | `transactions[].amount` — positive = outflow |

**Account owner:** omit — not in Plaid schema.

### Investment account detail — composite

Per-account [investment performance chart](examples/investment-account/investment-performance-chart.md) (filtered to one `account_id`), info block, segmented holdings bar (Stocks / Crypto / Cash), and activity flat table. See [investment account detail](examples/investment-account/investment-account-detail.md).

| Region | Source |
|---|---|
| Header | `title`, `institution_name`, `synced_at` |
| Performance chart | `chart.points[]`, `period_return_*` |
| Info block | `balances_available`, `subtype` |
| Holdings bar | `holdings_buckets[]` — `{ bucket, value, percent }` |
| Activities | `activities[]` — name, date, amount |

**Account owner:** omit — not in Plaid schema.
