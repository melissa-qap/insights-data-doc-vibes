# Monthly spending by category

### Description

Shows how much the user spent in the **current calendar month**, broken down by Plaid personal finance category, so they can see which categories drive the most outflows this month.

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `amount` | Transaction amount (positive = outflow, negative = inflow/refund) |
| `date` | Posted date — used in timeframe filter and calendar month assignment |
| `pending` | Whether transaction has posted — must be false |
| `personal_finance_category_primary` | Top-level spend category for grouping |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Month window**
   - Current calendar month only
   - Set `month` = `YYYY-MM` for today's date
   - `window_start` = first day of that month; `window_end` = last day of that month
2. **`as_of`**
   - First calendar day of that month (`YYYY-MM-01`)
3. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id`
4. **Account scope**
   - Keep rows where `account_type` in (`depository`, `credit`)
5. **Filter by timeframe**
   - Keep rows where `date` is on or between `window_start` and `window_end` (inclusive)
6. **Apply eligibility**
   - Exclude `personal_finance_category_primary` in (`INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, `LOAN_PAYMENTS`)
   - No amount sign filter (refunds net into category totals)
7. **Enrich**
   - `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
8. **Net against category**
   - For each transaction, sum `amount` into its `category` bucket
   - Plaid sign: positive = money out, negative = refund/credit; both count toward the category total (refunds reduce the category amount)
9. **Build flat rows**
   - One row per category: `{ month, category, amount }`
   - All rows share the same `month`
10. **Sort rows**
    - Order by `amount` descending (highest spend first)

### Data output

| Field | Type | Description |
|---|---|---|
| `month` | string | Current calendar month (`YYYY-MM`); same on every row |
| `rows` | array | Flat list: `{ month, category, amount }` — one row per primary category; sorted by `amount` descending |
| `as_of` | date | First calendar day of `month` (e.g. `2026-05-01` for May 2026) |
