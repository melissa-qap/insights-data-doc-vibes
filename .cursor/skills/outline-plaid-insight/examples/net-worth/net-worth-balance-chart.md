# Net worth balance chart

### Description

Shows the user's total net worth (assets minus liabilities) as a daily time series for a selected timeframe, with period dollar and percent change for that window. Suitable for a line chart of balance over time with a period change label.

Uses [net worth core](net-worth-core.md) with historical data access and totals-only output per day.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Unique account identifier |
| `type` | Account type (`depository`, `credit`, `investment`, `loan`, `other`) |
| `subtype` | Account subtype (e.g. `checking`, `credit card`) |
| `balances_current` | Current balance — used for net worth |
| `synced_at` | Snapshot timestamp; history retained per sync |

**Input:** `user_id = ?`. All account rows for the user (historical syncs, not latest-only). Exclude null `balances_current`.

Equivalent to [net worth core](net-worth-core.md) Layer 1 with historical query scope.

**Parameters:**

| Parameter | Values |
|---|---|
| `timeframe` | `trailing_1m`, `trailing_3m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |

**UI timeframe tab mapping:** `1M` → `trailing_1m`; `3M` → `trailing_3m`; `YTD` → `ytd`; `1Y` → `trailing_1y`; `All` → `all_time`.

### Calculation / analysis

1. **`window_end`**
   - Calendar date of `MAX(synced_at)` for the user ("today")
2. **Compute `window_start` from `timeframe`**
   - `trailing_1m` — 30 calendar days before `window_end`, inclusive
   - `trailing_3m` — 90 calendar days before `window_end`, inclusive
   - `trailing_6m` — 6 calendar months before `window_end`, inclusive
   - `ytd` — January 1 of `window_end`'s year
   - `trailing_1y` — 12 calendar months before `window_end`, inclusive
   - `all_time` — earliest calendar date on which any account has a row with `synced_at` on or before end of that day
3. **Daily net worth series**
   - For each calendar day `D` from `window_start` through `window_end`:
     1. **Resolve accounts** — [net worth core](net-worth-core.md) Layer 2: `resolve_accounts_as_of(user_id, end_of_day(D))`
     2. **Compute net worth** — [net worth core](net-worth-core.md) Layer 3: `compute_net_worth(accounts, include_groups = false)`
     3. Emit `{ date: D, net_worth }` from core output
     - If no account has a snapshot on or before `D`, omit `D` from the series (do not emit a point)
4. **Build `points`**
   - Collect daily `{ date, net_worth }` values
   - Sort ascending by `date`
5. **Derive period return from `points`**
   - Do not emit separate start/end or min/max fields — the UI derives hero value and Y-axis scale from `points[]` at render time
   - `period_return_amount` = `points[last].net_worth − points[0].net_worth`
   - `period_return_pct` = `period_return_amount / points[0].net_worth` when `points[0].net_worth > 0`; else omit
6. **`as_of`**
   - `window_end`

**Notes:**

- Between syncs, consecutive days share the same balance (carry forward until the next sync) — implemented by resolving at `end_of_day(D)` per [net worth core](net-worth-core.md).
- `all_time` begins on the first day with at least one account snapshot, not before linked accounts exist.
- **Return type:** holding-period change on account balances, not time-weighted return. Deposits, withdrawals, and transfers are not adjusted out.
- **Paired UI invariant:** `points[last].net_worth` must equal [net worth snapshot](net-worth-snapshot.md) `net_worth` when both use the same `window_end`.

### Data output

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | Requested window: `trailing_1m`, `trailing_3m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |
| `window_start` | date | First day in the filtered series |
| `window_end` | date | Last day in the filtered series |
| `points` | array | Chart series: `{ date, net_worth }`, sorted ascending by `date` |
| `period_return_amount` | number | `points[last].net_worth − points[0].net_worth` |
| `period_return_pct` | number | Fraction (e.g. `0.035` = 3.5%); omit if `points[0].net_worth` ≤ 0 |
| `as_of` | date | Same as `window_end` |

### UI output

**Pattern:** [Line chart](../../ui-output-options.md#net-worth-balance-chart--line-chart) — `date` on X-axis, `net_worth` on Y-axis; hero = `points[last].net_worth`; subtitle shows `period_return_amount` and `period_return_pct`. Y-axis scale derived by chart library from `points[]`. **`points[last].net_worth` is the net worth headline** when paired with [net worth snapshot](net-worth-snapshot.md) (that list omits a net worth total).
