# Net worth balance chart

### Description

Shows the user's total net worth (assets minus liabilities) as a daily time series for a selected timeframe, with period dollar and percent change for that window. Suitable for a line chart of balance over time with a period change label.

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

**Parameters:**

| Parameter | Values |
|---|---|
| `timeframe` | `trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |

### Calculation / analysis

1. Set `window_end` = calendar date of `MAX(synced_at)` for the user ("today").
2. Compute `window_start` from `timeframe`:
   - `trailing_1m` — 30 calendar days before `window_end`, inclusive
   - `trailing_6m` — 6 calendar months before `window_end`, inclusive
   - `ytd` — January 1 of `window_end`'s year
   - `trailing_1y` — 12 calendar months before `window_end`, inclusive
   - `all_time` — earliest calendar date on which any account has a row with `synced_at` on or before end of that day
3. For each calendar day `D` from `window_start` through `window_end`:
   - **Point-in-time accounts:** for each `account_id`, take the row with greatest `synced_at` where `synced_at <= end_of_day(D)`. Exclude null `balances_current`. Omit accounts with no row on or before `D`.
   - **Net worth for `D`** — apply [net worth snapshot](net-worth-snapshot.md) calculation steps 2–8 to that day's account set: classify each account, assign `group`, build the per-account list (`balance`, `role`, `group`), then derive category balances and `total_assets`, `total_liabilities`, and `net_worth` (category groups are optional for the chart; totals are required).
   - If no account has a snapshot on or before `D`, omit `D` from the series (do not emit a point).
4. Build `points` as `{ date: D, net_worth }` for each day with a computed value, sorted ascending by `date`.
5. **Derive from `points`** (do not compute independently):
   - `net_worth_min` = minimum `net_worth` in `points`; omit if `points` is empty
   - `net_worth_max` = maximum `net_worth` in `points`; omit if `points` is empty
   - `start_net_worth` = `points[0].net_worth`; `end_net_worth` = `points[last].net_worth`
   - `period_return_amount` = `end_net_worth - start_net_worth`
   - `period_return_pct` = `period_return_amount / start_net_worth` when `start_net_worth > 0`; else omit
6. Set `as_of` = `window_end`.

**Notes:**

- Between syncs, consecutive days share the same balance (carry forward until the next sync).
- `all_time` begins on the first day with at least one account snapshot, not before linked accounts exist.
- **Return type:** holding-period change on account balances, not time-weighted return. Deposits, withdrawals, and transfers are not adjusted out.

### Data output

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | Requested window: `trailing_1m`, `trailing_6m`, `ytd`, `trailing_1y`, `all_time` |
| `window_start` | date | First day in the filtered series |
| `window_end` | date | Last day in the filtered series |
| `points` | array | Chart series: `{ date, net_worth }`, sorted ascending by `date` |
| `net_worth_min` | number | Minimum `net_worth` in `points` (derived from `points`) |
| `net_worth_max` | number | Maximum `net_worth` in `points` (derived from `points`) |
| `start_net_worth` | number | `points[0].net_worth` |
| `end_net_worth` | number | `points[last].net_worth` |
| `period_return_amount` | number | `end_net_worth - start_net_worth` |
| `period_return_pct` | number | Fraction (e.g. `0.035` = 3.5%); omit if `start_net_worth` ≤ 0 |
| `as_of` | date | Same as `window_end` |

### UI output

**Pattern:** [Line chart](../../ui-output-options.md#net-worth-balance-chart--line-chart) — `date` on X-axis, `net_worth` on Y-axis; Y-axis domain `[net_worth_min, net_worth_max]`; hero shows `end_net_worth`; subtitle shows `period_return_amount` and `period_return_pct`. **`end_net_worth` is the net worth headline** when paired with [net worth snapshot](net-worth-snapshot.md) (that list omits a net worth total).
