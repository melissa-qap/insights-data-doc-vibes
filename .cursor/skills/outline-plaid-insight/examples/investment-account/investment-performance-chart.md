# Investment performance chart

### Description

Shows combined investment account value as a daily time series for a selected timeframe, with simple holding-period return ($ and %) for that window. Suitable for a line chart of portfolio value over time with a period performance label.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Unique account identifier |
| `type` | Account type — must be `investment` |
| `balances_current` | Current account balance — used for total value |
| `synced_at` | Snapshot timestamp; history retained per sync |

**Input:** `user_id = ?`. All historical rows for the user (not latest-only). `type = investment`. Exclude null `balances_current`.

**Parameters:**

| Parameter | Values | Default |
|---|---|---|
| `timeframe` | `trailing_1m`, `trailing_3m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` | `trailing_6m` |

### Calculation / analysis

1. **`window_end`**
   - Calendar date of `MAX(synced_at)` for the user ("today")
2. **Compute `window_start` from `timeframe`**
   - `trailing_1m` — 30 calendar days before `window_end`, inclusive
   - `trailing_3m` — 90 calendar days before `window_end`, inclusive
   - `trailing_6m` — 6 calendar months before `window_end`, inclusive
   - `ytd` — January 1 of `window_end`'s year
   - `trailing_1y` — 12 calendar months before `window_end`, inclusive
   - `all_time` — earliest calendar date on which any investment account has a row with `synced_at` on or before end of that day
3. **Daily portfolio value series**
   - For each calendar day `D` from `window_start` through `window_end`:
     - **Point-in-time accounts:** for each `account_id` where `type = investment`, take the row with greatest `synced_at` where `synced_at <= end_of_day(D)`. Exclude null `balances_current`. Omit accounts with no row on or before `D`.
     - Use `balances_current` only (not `balances_available`)
     - `total_value` = sum of `balances_current` across included investment accounts for that day
     - If no investment account has a snapshot on or before `D`, omit `D` from the series (do not emit a point)
4. **Build `points`**
   - `{ date: D, total_value }` for each day with a computed value
   - Sort ascending by `date`
5. **Derive from `points`**
   - Do not compute independently
   - `total_value_min` = minimum `total_value` in `points`; omit if `points` is empty
   - `total_value_max` = maximum `total_value` in `points`; omit if `points` is empty
   - `start_value` = `points[0].total_value`; `end_value` = `points[last].total_value`
   - `period_return_amount` = `end_value - start_value`
   - `period_return_pct` = `period_return_amount / start_value` when `start_value > 0`; else omit
6. **`as_of`**
   - `window_end`

**Notes:**

- Between syncs, consecutive days share the same balance (carry forward until the next sync).
- `all_time` begins on the first day with at least one investment account snapshot.
- **Return type:** holding-period return on account balances, not time-weighted return. Deposits, withdrawals, and transfers are not adjusted out.
- **Not available in current Plaid schema:** benchmark overlay, time-weighted / money-weighted return, per-security performance lines, historical return from `cost_basis`.

### Data output

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | Requested window: `trailing_1m`, `trailing_3m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |
| `window_start` | date | First day in the filtered series |
| `window_end` | date | Last day in the filtered series |
| `points` | array | Chart series: `{ date, total_value }`, sorted ascending by `date` |
| `total_value_min` | number | Minimum `total_value` in `points` (derived from `points`) |
| `total_value_max` | number | Maximum `total_value` in `points` (derived from `points`) |
| `start_value` | number | `points[0].total_value` |
| `end_value` | number | `points[last].total_value` |
| `period_return_amount` | number | `end_value - start_value` |
| `period_return_pct` | number | Fraction (e.g. `0.035` = 3.5%); omit if `start_value` ≤ 0 |
| `as_of` | date | Same as `window_end` |

### UI output

**Pattern:** [Line chart](../../ui-output-options.md#investment-performance-chart--line-chart) — `date` on X-axis, `total_value` on Y-axis; Y-axis domain `[total_value_min, total_value_max]`; subtitle shows `period_return_amount` and `period_return_pct`. **`end_value` is the portfolio headline total** when paired with [investment accounts by institution](investment-accounts-by-institution.md) (that list omits a root total).
