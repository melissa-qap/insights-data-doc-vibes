# Cash inflow and outflow chart

### Description

Shows monthly money coming in vs going out over the trailing 24 months on linked depository and credit accounts, with net cash flow per month so users can see whether they ran a surplus or deficit each month.

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date — used in timeframe filter and calendar month assignment |
| `pending` | Whether transaction has posted — must be false |
| `personal_finance_category_primary` | Top-level category — used in eligibility filter |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id`
2. **Month window**
   - `window_end` = calendar date of `MAX(date)` on loaded table (or lightweight pre-query)
   - `window_start` = first day of the calendar month that is 23 months before `window_end`'s month (24 calendar months inclusive, e.g. Jun 2024–May 2026 when latest transaction is in May 2026)
3. **Account scope**
   - Keep rows where `account_type` in (`depository`, `credit`)
4. **Filter by timeframe**
   - Keep rows where `date` is on or between `window_start` and `window_end` (inclusive)
5. **Apply eligibility**
   - Exclude `personal_finance_category_primary` in (`TRANSFER_IN`, `TRANSFER_OUT`)
   - No amount sign filter
6. **Per month**
   - Derive `month` as `YYYY-MM` from `date`
   - For each month:
     - `window_start` = first calendar day of `month`
     - `window_end` = last calendar day of `month`; when `month` is the same calendar month as top-level `window_end`, use top-level `window_end` instead (partial latest month)
     - `cash_inflow` = sum of `ABS(amount)` where `amount < 0` (positive magnitude for upward bar)
     - `cash_outflow` = −sum of `amount` where `amount > 0` (negative magnitude for downward bar)
     - `net_cash_flow` = `cash_inflow + cash_outflow` (derived from bar values; equals −sum of `amount` for the month)
7. **Build `months`**
   - One object per month in the window: `{ month, window_start, window_end, cash_inflow, cash_outflow, net_cash_flow }`
   - Include months with zero activity (all cash fields `0`; still set `window_start` / `window_end` from `month`)
   - Sort ascending by `month`
8. **Derive chart bounds from `months`**
   - Do not compute independently
   - `value_min` = minimum of `cash_outflow` and `net_cash_flow` across all months; omit if `months` is empty
   - `value_max` = maximum of `cash_inflow` and `net_cash_flow` across all months; omit if `months` is empty
9. **`as_of`**
   - `window_end`

### Data output

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | `"trailing_24m"` |
| `window_start` | date | First day of earliest month in series |
| `window_end` | date | Last day of latest month in series |
| `months` | array | Chart series: up to 24 `{ month, window_start, window_end, cash_inflow, cash_outflow, net_cash_flow }`, sorted ascending by `month` |
| `value_min` | number | Scale floor — derived from all `months` (stable across viewport slices) |
| `value_max` | number | Scale ceiling — derived from all `months` (stable across viewport slices) |
| `as_of` | date | Same as `window_end` |
