# UI output options

How insight **data output** maps to UI presentation patterns. Each insight data spec in [examples/](examples/README.md) may reference one or more UI options from this file.

**Convention:** Every UI node has a **label** (what it is) and a **balance** (the correlated number at that level). Parent balances are rollups of their children unless noted otherwise.

---

## Patterns index

| Insight | UI pattern | Section |
|---|---|---|
| Net worth snapshot | Nested list | [Net worth — nested list](#net-worth--nested-list) |
| Net worth balance chart | Line chart | [Net worth balance chart — line chart](#net-worth-balance-chart--line-chart) |
| Monthly spending by category | Flat table | [Monthly spending by category — flat table](#monthly-spending-by-category--flat-table) |
| Recurring spending | Flat table | [Recurring spending — flat table](#recurring-spending--flat-table) |
| Top 5 holdings | Nested list | [Top 5 holdings — nested list](#top-5-holdings--nested-list) |
| Recurring investments | Flat table | [Recurring investments — flat table](#recurring-investments--flat-table) |
| Asset allocation | Nested list | [Asset allocation — nested list](#asset-allocation--nested-list) |
| Investment performance chart | Line chart | [Investment performance chart — line chart](#investment-performance-chart--line-chart) |
| Investment accounts by institution | Nested list | [Investment accounts by institution — nested list](#investment-accounts-by-institution--nested-list) |
| Cash inflow and outflow chart | Combo line + bar chart | [Cash inflow/outflow chart — combo line and bar chart](#cash-inflow-outflow-chart--combo-line-and-bar-chart) |

---

## Net worth — nested list

A four-level breakdown: Assets and Liabilities as top-level section headers (no balance), category groups (Cash, Investment, Credit cards, Loans) with balances, individual accounts at the leaves. **No net worth total in this list** — show combined net worth from [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) (`end_net_worth`) in the same screen (e.g. chart header or hero above the list).

### Hierarchy

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

### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Section | "Assets" | — (omit) | — |
| Group | "Cash" | `cash_balance` | `cash_balance`, `asset_groups[]` |
| Group | "Investment" | `investment_balance` | `investment_balance`, `asset_groups[]` |
| Leaf | `account.name` | `account.balance` | `asset_groups[].accounts[]` |
| Section | "Liabilities" | — (omit) | — |
| Group | "Credit cards" | `credit_cards_balance` | `credit_cards_balance`, `liability_groups[]` |
| Group | "Loans" | `loans_balance` | `loans_balance`, `liability_groups[]` |
| Leaf | `account.name` | `account.balance` | `liability_groups[].accounts[]` |

### Build steps

1. Start from the [net worth snapshot data output](examples/net-worth/net-worth-snapshot.md) (`asset_groups`, `liability_groups`, category balances, `as_of`).
2. Render Assets and Liabilities as section headers (no balance column value) → category groups → account leaves.
3. Render asset category groups in order: Cash, Investment. Liability category groups: Credit cards, Loans. Skip groups with no accounts.
4. Verify category and account balances; each group `total_balance` = sum of child `balance`.
5. **Net worth total (sibling widget):** load [net worth balance chart](examples/net-worth/net-worth-balance-chart.md) for the same user/timeframe; display `end_net_worth` (and optional `period_return_amount` / `period_return_pct` from the chart subtitle) above or beside this list — do not show `net_worth` from the snapshot payload as a list root row.
6. Optional footer on the list: "As of {as_of}" from snapshot `as_of` (align with chart `as_of` when both use the active window end).

### Example (rendered)

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

---

## Monthly spending by category — flat table

A single-level table for the **current calendar month** only: one row per category. No nesting, no per-month total row, and no `Month` column (month is in the subtitle).

### Columns

| Column | Source |
|---|---|
| Category | `row.category` (`personal_finance_category_primary`) |
| Amount | `row.amount` (net spend for that category in the current month) |

### Build steps

1. Start from the [monthly spending data output](examples/cash-flow/monthly-spending-by-category.md) (`month`, `rows`, `as_of`).
2. Render `rows` in order (highest `amount` first). All rows are for `month`.
3. Subtitle: current month label from `month` or `as_of` (e.g. "May 2026 · as of May 1, 2026").

### Example (rendered)

*May 2026 · as of May 1, 2026*

| Category | Amount |
|---|---|
| FOOD_AND_DRINK | $412.00 |
| TRANSPORTATION | $186.50 |
| GENERAL_MERCHANDISE | $95.20 |

### Notes

- Data is scoped to the current calendar month only — do not show prior months in this table.
- Categories are ordered by amount descending (highest spend at top).
- Category labels: map Plaid codes (e.g. `FOOD_AND_DRINK`) to display names in the UI layer.
- Negative net category amounts (refunds exceeded spend) display as negative values in the Amount column.
- `as_of` is the first calendar day of the current month (`YYYY-MM-01`), not the latest transaction date.

---

## Top 5 holdings — nested list

A two-level hierarchy: holding (security) at the top with total value, investment accounts nested below with each account's portion. Up to five holding groups, ordered by `total_value` descending.

### Hierarchy

```
1. {security name} ({ticker})          → holding.total_value
   ├── {account name}                  → account.value
   └── ...
2. ...
```

### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Holding (root per group) | `holding.name` (+ ticker) | `holding.total_value` | `holdings[]` |
| Account (child) | `account.account_name` | `account.value` | `holdings[].accounts[]` |

### Build steps

1. Start from the [top 5 holdings data output](examples/investment-account/top-5-holdings.md) (`holdings`, `as_of`).
2. Render each holding in `rank` order (1–5).
3. Under each holding, render `accounts` in `value` descending order.
4. Verify `holding.total_value` = sum of `accounts[].value`.
5. Show `as_of` as subtitle or footnote.

### Example (rendered)

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

### Notes

- Display rank prefix (1–5) on holding rows only, not on account children.
- Display ticker in parentheses when both `name` and `ticker_symbol` are present.
- Single-account holdings: show one child row (no flattening to a flat list).
- Fewer than 5 holdings: show only available holding groups.
- Tie-breaking when `total_value` is equal: sort holdings by `name` alphabetically.

---

## Recurring investments — flat table

One row per active recurring investment pattern, sorted by frequency then amount.

### Columns

| Column | Source |
|---|---|
| Account | `recurrence.account_name` |
| Security | `recurrence.security_name` (+ `ticker_symbol` if present) |
| Frequency | `recurrence.frequency` (`weekly`, `biweekly`, `semi-monthly`, `monthly`, `quarterly`, `annual`) |
| Amount | `recurrence.typical_amount` |
| Last date | `recurrence.last_date` |

### Build steps

1. Start from the [recurring investments data output](examples/investment-account/recurring-investments.md) (`recurrences`, `window_start`, `window_end`).
2. Render each item in `recurrences` as one table row (pre-sorted).
3. Optional footer: "Patterns from {window_start} to {window_end}".

### Example (rendered)

| Account | Security | Frequency | Amount | Last date |
|---|---|---|---|---|
| Fidelity 401k | Vanguard Target 2055 (VTTVX) | Monthly | $500.00 | 2026-05-01 |
| Robinhood | Apple Inc. (AAPL) | Biweekly | $100.00 | 2026-05-20 |
| Fidelity 401k | Vanguard Total Stock (VTI) | Semi-monthly | $250.00 | 2026-05-15 |

*Patterns detected from Nov 26, 2025 to May 26, 2026*

### Notes

- Map `frequency` to display labels (e.g. `semi-monthly` → "Semi-monthly").
- Optional column **Occurrences** from `occurrence_count` for power users.

---

## Recurring spending — flat table

One row per active recurring spend pattern, sorted by frequency then amount.

### Columns

| Column | Source |
|---|---|
| Merchant | `recurrence.merchant_name` |
| Category | `recurrence.category` (optional column) |
| Frequency | `recurrence.frequency` (`weekly`, `biweekly`, `semi-monthly`, `monthly`, `quarterly`, `annual`) |
| Amount | `recurrence.typical_amount` |
| Last date | `recurrence.last_date` |

### Build steps

1. Start from the [recurring spending data output](examples/cash-flow/recurring-spending.md) (`recurrences`, `window_start`, `window_end`).
2. Render each item in `recurrences` as one table row (pre-sorted).
3. Optional footer: "Patterns detected from {window_start} to {window_end}".

### Example (rendered)

| Merchant | Category | Frequency | Amount | Last date |
|---|---|---|---|---|
| Netflix | ENTERTAINMENT | Monthly | $15.99 | 2026-05-01 |
| Spotify | ENTERTAINMENT | Monthly | $11.99 | 2026-05-03 |
| PG&E | RENT_AND_UTILITIES | Monthly | $142.50 | 2026-04-28 |

*Patterns detected from Nov 26, 2025 to May 26, 2026*

### Notes

- Map `frequency` to display labels (e.g. `semi-monthly` → "Semi-monthly").
- Optional column **Occurrences** from `occurrence_count` for power users.
- **Category** can be hidden in compact layouts.

---

## Asset allocation — nested list

A two-level hierarchy: portfolio allocation at the root, asset-class groups in the middle, individual securities at the leaves. Class balances are rollups of attributed holding values; the root shows `total_investment_value`. Unclassified positions appear as an **Other** class row with nested holdings — not as a footnote. Asset-class rows support toggling between **percent** and **dollar** display; holding leaves always show dollars.

### Display mode (% / $)

| Mode | Class row shows | Root shows | Holding rows show |
|---|---|---|---|
| **$** (default) | `class.value` formatted as currency | `total_investment_value` | `holding.value` |
| **%** | `class.percent` formatted as percent (e.g. `percent × 100` → `42%`) | `total_investment_value` (unchanged) | `holding.value` (unchanged) |

- Provide a single control (segmented toggle, icon button, etc.) that switches all class rows between modes. Persist the user's choice for the session when possible.
- Show **one** metric per class row — do not show both percent and dollar on the same row.
- `percent` is a fraction (0–1) in data output; multiply by 100 for display. Round displayed percents to whole numbers unless the UI needs decimals for small slices.

### Hierarchy

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

### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Root | "Portfolio allocation" | `total_investment_value` | `total_investment_value` |
| Class (child) | Display label for `asset_class` (`other` → "Other") | `class.value` ($ mode) or `class.percent` (% mode) | `asset_classes[]` |
| Holding (leaf) | `holding.name` (+ ticker) | `holding.value` | `asset_classes[].holdings[]` |

### Build steps

1. Start from the [asset allocation data output](examples/investment-account/asset-allocation.md) (`asset_classes`, `total_investment_value`, `as_of`).
2. Create root → class groups → holding leaves in `asset_classes` order (`value` descending).
3. Render each class row using the active display mode (`value` or `percent`), including **Other** when `asset_class = other`.
4. Under **Other**, render `holdings` in `value` descending order (same as other classes).
5. Verify `class.value` = sum of `holdings[].value` for that class; verify displayed percent matches `class.percent`.
6. Verify root `total_investment_value` = sum of all `asset_classes[].value`.
7. Show `as_of` as subtitle or footnote only — do not use a separate unclassified footnote.

### Example (rendered)

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

### Notes

- Map `asset_class` codes to display labels in the UI layer (e.g. `international_equity` → "International equities", `other` → "Other").
- **Other** holds securities missing fund-composition enrichment; omit the class row when `unclassified_value` is 0.
- Display ticker in parentheses when both `name` and `ticker_symbol` are present.
- A security may appear under multiple classes when enrichment splits an ETF across classes.
- Omit asset classes with zero `value`.
- Tie-breaking when `value` is equal: sort classes and holdings by display label alphabetically.
- Class percents are shares of `total_investment_value`; all class rows (including **Other**) sum to 100% in % mode.

---

## Net worth balance chart — line chart

A time-series line chart: one point per calendar day showing total net worth. The X-axis is date; the Y-axis is net worth balance. A subtitle shows period dollar and percent change for the active timeframe.

### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Net worth" |
| Hero value | `end_net_worth` | Currency (e.g. `$122,500`) — latest net worth at `as_of` |
| Subtitle — dollar change | `period_return_amount` | Signed currency (e.g. `+$4,300`) |
| Subtitle — percent change | `period_return_pct` | Signed percent (`period_return_pct × 100`, e.g. `+3.6%`) |
| Subtitle — window | `timeframe`, `as_of` | Human label (e.g. "Trailing 6 months · as of May 26, 2026") |

Example header: **$122,500** · `+$4,300 (+3.6%) · Trailing 6 months · as of May 26, 2026`

Omit percent segment when `period_return_pct` is absent (`start_net_worth` ≤ 0).

### Axes

| Axis | Source | Format |
|---|---|---|
| X | `point.date` | Date (e.g. May 26 or 05/26) |
| Y | `point.net_worth` | Currency |
| Y domain (min) | `net_worth_min` | Lower bound of Y-axis for this timeframe |
| Y domain (max) | `net_worth_max` | Upper bound of Y-axis for this timeframe |

Set the Y-axis range to `[net_worth_min, net_worth_max]` — the min and max `net_worth` values in `points` for the selected timeframe. Do not pad to zero or use a fixed global scale.

### Build steps

1. Start from the [net worth balance chart data output](examples/net-worth/net-worth-balance-chart.md) (`points`, `net_worth_min`, `net_worth_max`, `period_return_amount`, `period_return_pct`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Render title and subtitle from header table above.
3. Map each item in `points` to a chart coordinate: `(date, net_worth)`.
4. Set chart Y-axis domain to `net_worth_min` (bottom) and `net_worth_max` (top). Verify they match `MIN(points[].net_worth)` and `MAX(points[].net_worth)`.
5. Connect points in array order (ascending `date`).
6. Format Y-axis labels as currency; choose X-axis tick density based on `timeframe` (e.g. weekly ticks for `trailing_1m`, monthly for `trailing_1y`).
7. Optional **timeframe** control (`trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time`): re-run the insight with the selected `timeframe` and re-render (updates `points`, bounds, and return fields).

### Example (rendered)

| Date | Net worth |
|---|---|
| 2025-11-26 | $118,200 |
| 2025-12-01 | $119,400 |
| 2026-01-15 | $120,800 |
| 2026-03-01 | $121,500 |
| 2026-05-01 | $122,100 |
| 2026-05-26 | $122,500 |

*+$4,300 (+3.6%) · Trailing 6 months · as of May 26, 2026*

### Notes

- Y-axis range is always scoped to the active timeframe — when `timeframe` changes, recompute domain from the new `net_worth_min` / `net_worth_max`.
- When all values are equal (`net_worth_min` = `net_worth_max`), use that single value for both bounds (flat line); add minimal visual padding only if the chart library requires it.
- `points` must be sorted ascending by `date`; do not re-sort in the UI layer.
- Single point: render as a dot or short flat segment; period return is zero.
- Period return is simple holding-period change on net worth balances, not time-weighted return.
- Empty `points`: show an empty state (no linked accounts or no history in range).
- Between syncs, flat segments are expected (carry-forward balances).
- Drill-down to per-account balances: use [net worth snapshot](examples/net-worth/net-worth-snapshot.md), not this chart payload.
- **Net worth headline:** On net worth overview screens, show `end_net_worth` (and period return from the subtitle) as the primary total; pair with [net worth snapshot](examples/net-worth/net-worth-snapshot.md) for the account breakdown (no net worth root row on the list).

---

## Investment accounts by institution — nested list

A two-level hierarchy: institutions at the top, individual investment accounts nested below. Institution balances are rollups of child account balances. **No portfolio total in this list** — show combined value from [investment performance chart](examples/investment-account/investment-performance-chart.md) (`end_value`) in the same screen (e.g. chart header or hero above the list).

### Hierarchy

```
{institution_name}                       → institution.total_balance
├── {account name}                       → account.balance
└── ...
```

### Node mapping

| UI level | Label | Balance source | Data fields |
|---|---|---|---|
| Group | `institution.institution_name` | `institution.total_balance` | `institutions[]` |
| Leaf | `account.name` (+ optional ` ••{mask}`) | `account.balance` | `institutions[].accounts[]` |

### Build steps

1. Start from the [investment accounts by institution data output](examples/investment-account/investment-accounts-by-institution.md) (`institutions`, `as_of`).
2. Render institution groups in `total_balance` descending order; nest `accounts` under each.
3. Within each institution, render `accounts` in `balance` descending order.
4. Verify `institution.total_balance` = sum of `accounts[].balance`.
5. **Portfolio total (sibling widget):** load [investment performance chart](examples/investment-account/investment-performance-chart.md) for the same user/timeframe; display `end_value` (and optional `period_return_amount` / `period_return_pct`) above or beside this list — do not sum institution rows for the headline total.
6. Optional footer on the list: "As of {as_of}" from `as_of` (align with chart `as_of` when both use latest snapshot).

### Example (rendered)

*Portfolio total from investment performance chart: **$186,600** (+3.5% trailing 6m)*

```
Fidelity                     $120,000
├── 401k Plan                 $95,000
└── Brokerage ••4821          $25,000
Vanguard                      $66,600
├── Traditional IRA           $48,200
└── Roth IRA                  $18,400
```

### Notes

- Append ` ••{mask}` to the account label when `mask` is present.
- Show `subtype` as a secondary label or badge (e.g. "401k", "ira") when useful for disambiguation.
- `balance_source = holdings` accounts: optional subtle indicator that balance was derived from positions (institution did not report account-level balance).
- Single account under one institution: one parent row with one child (no flattening to a flat list).
- Empty `institutions`: show an empty state (no linked investment accounts).
- Drill-down to holdings: use [top 5 holdings](examples/investment-account/top-5-holdings.md) or [asset allocation](examples/investment-account/asset-allocation.md).

---

## Investment performance chart — line chart

A time-series line chart: one point per calendar day showing combined investment account value. The X-axis is date; the Y-axis is total value. A subtitle shows period dollar and percent return for the active timeframe.

### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Investment performance" |
| Subtitle — dollar change | `period_return_amount` | Signed currency (e.g. `+$4,200`) |
| Subtitle — percent change | `period_return_pct` | Signed percent (`period_return_pct × 100`, e.g. `+3.5%`) |
| Subtitle — window | `timeframe`, `window_end` | Human label (e.g. "Trailing 6 months · as of May 26, 2026") |

Example subtitle: `+$4,200 (+3.5%) · Trailing 6 months · as of May 26, 2026`

Omit percent segment when `period_return_pct` is absent (`start_value` ≤ 0).

### Axes

| Axis | Source | Format |
|---|---|---|
| X | `point.date` | Date (e.g. May 26 or 05/26) |
| Y | `point.total_value` | Currency |
| Y domain (min) | `total_value_min` | Lower bound of Y-axis for this timeframe |
| Y domain (max) | `total_value_max` | Upper bound of Y-axis for this timeframe |

Set the Y-axis range to `[total_value_min, total_value_max]` — the min and max `total_value` values in `points` for the selected timeframe. Do not pad to zero or use a fixed global scale.

### Build steps

1. Start from the [investment performance chart data output](examples/investment-account/investment-performance-chart.md) (`points`, `total_value_min`, `total_value_max`, `period_return_amount`, `period_return_pct`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Render title and subtitle from header table above.
3. Map each item in `points` to a chart coordinate: `(date, total_value)`.
4. Set chart Y-axis domain to `total_value_min` (bottom) and `total_value_max` (top). Verify they match `MIN(points[].total_value)` and `MAX(points[].total_value)`.
5. Connect points in array order (ascending `date`).
6. Format Y-axis labels as currency; choose X-axis tick density based on `timeframe` (e.g. weekly ticks for `trailing_1m`, monthly for `trailing_1y`).
7. Optional **timeframe** control (`trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time`): re-run the insight with the selected `timeframe` and re-render (updates `points`, bounds, and return fields).

### Example (rendered)

| Date | Total value |
|---|---|
| 2025-11-26 | $82,400 |
| 2025-12-01 | $83,100 |
| 2026-01-15 | $84,500 |
| 2026-03-01 | $85,200 |
| 2026-05-01 | $86,100 |
| 2026-05-26 | $86,600 |

*+$4,200 (+5.1%) · Trailing 6 months · as of May 26, 2026*

### Notes

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

## Cash inflow/outflow chart — combo line and bar chart

A layered combo chart: grouped bars per calendar month for cash inflow (positive) and cash outflow (negative), with a line overlay for net cash flow. The X-axis is month; the Y-axis is currency. An optional subtitle shows period net cash flow for the trailing 6 months.

### Header

| Element | Source | Format |
|---|---|---|
| Title | Static | "Cash flow" |
| Subtitle — period net | `period_net_cash_flow` | Signed currency (e.g. `+$1,200` over 6 months) |
| Subtitle — window | `timeframe`, `as_of` | Human label (e.g. "Trailing 6 months · as of May 26, 2026") |

Example subtitle: `+$1,200 net · Trailing 6 months · as of May 26, 2026`

Omit subtitle when `months` is empty.

### Axes

| Axis | Source | Format |
|---|---|---|
| X | `month.month` | Month label (e.g. `Dec 2025` or `12/25`) |
| Y — bars | `cash_inflow`, `cash_outflow` | Currency; inflow above zero, outflow below zero |
| Y — line | `net_cash_flow` | Currency; same scale as bars |
| Y domain (min) | `value_min` | Lower bound — min of `cash_outflow` and `net_cash_flow` across months |
| Y domain (max) | `value_max` | Upper bound — max of `cash_inflow` and `net_cash_flow` across months |

Set the Y-axis range to `[value_min, value_max]`. Include zero on the axis when it falls within the domain. Do not pad to a fixed global scale.

### Series

| Series | Type | Source | Visual |
|---|---|---|---|
| Inflow | Bar | `cash_inflow` | Positive bar above zero (e.g. green) |
| Outflow | Bar | `cash_outflow` | Negative bar below zero (e.g. red) |
| Net cash flow | Line | `net_cash_flow` | Line/markers overlaid on bars, distinct color |

Group inflow and outflow bars by `month` on the X-axis (side-by-side or overlapping per library). Draw the net line on top of the bar layer.

### Build steps

1. Start from the [cash inflow and outflow chart data output](examples/cash-flow/cash-inflow-outflow-chart.md) (`months`, `value_min`, `value_max`, `period_net_cash_flow`, `window_start`, `window_end`, `timeframe`, `as_of`).
2. Render title and optional subtitle from header table above.
3. For each item in `months`, render bars at `(month, cash_inflow)` and `(month, cash_outflow)`.
4. Map each item in `months` to a line coordinate: `(month, net_cash_flow)`; connect in array order (ascending `month`).
5. Set chart Y-axis domain to `value_min` (bottom) and `value_max` (top). Verify they match `MIN(months[].cash_outflow, months[].net_cash_flow)` and `MAX(months[].cash_inflow, months[].net_cash_flow)`.
6. Add a horizontal zero reference line when zero is within `[value_min, value_max]`.
7. Format Y-axis labels as currency; use one tick per month on the X-axis for `trailing_6m`.
8. Legend: **Inflow**, **Outflow**, **Net cash flow**.

### Example (rendered)

| Month | Inflow | Outflow | Net |
|---|---|---|---|
| 2025-12 | $8,200 | −$6,400 | +$1,800 |
| 2026-01 | $8,100 | −$7,200 | +$900 |
| 2026-02 | $8,000 | −$8,500 | −$500 |
| 2026-03 | $8,300 | −$6,900 | +$1,400 |
| 2026-04 | $8,150 | −$7,100 | +$1,050 |
| 2026-05 | $8,400 | −$6,800 | +$1,600 |

*+$5,250 net · Trailing 6 months · as of May 26, 2026*

### Notes

- Y-axis range is scoped to the active `months` payload — recompute `value_min` / `value_max` when data refreshes.
- When all values are equal (`value_min` = `value_max`), use that single value for both bounds (flat chart); add minimal visual padding only if the chart library requires it.
- `months` must be sorted ascending by `month`; do not re-sort in the UI layer.
- Empty `months`: show an empty state (no linked accounts or no transactions in range).
- Months with no transactions still appear with zero bars and zero net (per data spec).
- Drill-down to category spend: use [monthly spending by category](examples/cash-flow/monthly-spending-by-category.md), not this chart payload.
