# Cash inflow and outflow chart

### Description

Shows monthly money coming in vs going out over the last six months on linked depository and credit accounts, with a net cash flow line so users can see whether they ran a surplus or deficit each month.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier |
| `type` | Account type — must be `depository` or `credit` |

**Input:** `user_id = ?`. `type` in (`depository`, `credit`). Used to scope transactions to cash-flow accounts.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date — used to assign calendar month |
| `pending` | Whether transaction has posted — must be false |
| `personal_finance_category_primary` | Top-level category — used to exclude transfers |

**Input:** `user_id = ?`. `account_id` in depository/credit account list. `date` within month window. `pending = false`. Exclude `transaction_id` in `plaid_transactions_removed`.

#### `plaid_transactions_removed`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | ID of transaction removed from active set |

**Input:** `user_id = ?`. Used to build exclusion list for active transactions.

### Calculation / analysis

1. **Account scope** — load `account_id` from `plaid_accounts` where `type` in (`depository`, `credit`).
2. **Month window** — set `window_end` = calendar date of `MAX(date)` among active scoped transactions. Set `window_start` = first day of the calendar month that is 5 months before `window_end`'s month (6 calendar months inclusive, e.g. Dec–May when latest transaction is in May).
3. Load active transactions for scoped accounts where `date` is on or between `window_start` and `window_end`. Exclude `pending = true` and IDs in `plaid_transactions_removed`.
4. **Category filter** — exclude transactions where `personal_finance_category_primary` is `TRANSFER_IN` or `TRANSFER_OUT`. Keep `INCOME`, spend categories, `LOAN_PAYMENTS`, and null/uncategorized.
5. **Per month** — derive `month` as `YYYY-MM` from `date`. For each month:
   - `cash_inflow` = sum of `ABS(amount)` where `amount < 0` (positive magnitude for upward bar)
   - `cash_outflow` = −sum of `amount` where `amount > 0` (negative magnitude for downward bar)
   - `net_cash_flow` = `cash_inflow + cash_outflow` (derived from bar values; equals −sum of `amount` for the month)
6. **Build `months`** — one object per month in the window: `{ month, cash_inflow, cash_outflow, net_cash_flow }`. Include months with zero activity (all fields `0`). Sort ascending by `month`.
7. **Derive chart bounds from `months`** (do not compute independently):
   - `value_min` = minimum of `cash_outflow` and `net_cash_flow` across all months; omit if `months` is empty
   - `value_max` = maximum of `cash_inflow` and `net_cash_flow` across all months; omit if `months` is empty
8. **Derive period rollup from `months`** — `period_net_cash_flow` = sum of `net_cash_flow` across months.
9. Set `as_of` = `window_end`.

### Data output

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | `"trailing_6m"` |
| `window_start` | date | First day of earliest month in series |
| `window_end` | date | Last day of latest month in series |
| `months` | array | Chart series: `{ month, cash_inflow, cash_outflow, net_cash_flow }`, sorted ascending by `month` |
| `value_min` | number | Y-axis floor — derived from `months` |
| `value_max` | number | Y-axis ceiling — derived from `months` |
| `period_net_cash_flow` | number | Sum of `net_cash_flow` across `months` (derived) |
| `as_of` | date | Same as `window_end` |

### UI output

**Pattern:** [Combo line and bar chart](../../ui-output-options.md#cash-inflow-outflow-chart--combo-line-and-bar-chart) — `month` on X-axis; `cash_inflow` and `cash_outflow` as bars; `net_cash_flow` as overlaid line; Y-axis domain `[value_min, value_max]`; optional subtitle shows `period_net_cash_flow`.
