# Monthly spending by category

### Description

Shows how much the user spent in the **current calendar month**, broken down by Plaid personal finance category, so they can see which categories drive the most outflows this month.

### Required input data

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `amount` | Transaction amount (positive = outflow, negative = inflow/refund) |
| `date` | Posted date — used to assign calendar month |
| `pending` | Whether transaction has posted — must be false |
| `personal_finance_category_primary` | Top-level spend category for grouping |

**Input:** `user_id = ?`. `date` on or between the first and last day of the **current calendar month** (inclusive). `pending = false`. Exclude `transaction_id` in `plaid_transactions_removed`.

#### `plaid_transactions_removed`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | ID of transaction removed from active set |

**Input:** `user_id = ?`. Used to build exclusion list for active transactions.

### Calculation / analysis

1. **Month window:** Current calendar month only. Set `month` = `YYYY-MM` for today’s date. Include transactions where `date` falls on or between the first and last day of that month.
2. Set `as_of` = first calendar day of that month (`YYYY-MM-01`).
3. Load active transactions for the user in that month.
4. **Category filter** — exclude transactions where `personal_finance_category_primary` is `INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, or `LOAN_PAYMENTS`. Use primary category only (not `personal_finance_category_detailed`).
5. **Net against category** — for each remaining transaction, sum `amount` into its primary category bucket. Plaid sign: positive = money out, negative = refund/credit; both count toward the category total (refunds reduce the category amount).
6. **Build flat rows** — one row per `personal_finance_category_primary`: `{ month, category, amount }`. All rows share the same `month`. Use `"UNCATEGORIZED"` when `personal_finance_category_primary` is null.
7. **Sort rows** — order by `amount` descending (highest spend first).

### Data output

| Field | Type | Description |
|---|---|---|
| `month` | string | Current calendar month (`YYYY-MM`); same on every row |
| `rows` | array | Flat list: `{ month, category, amount }` — one row per primary category; sorted by `amount` descending |
| `as_of` | date | First calendar day of `month` (e.g. `2026-05-01` for May 2026) |

### UI output

**Pattern:** [Flat table](../../ui-output-options.md#monthly-spending-by-category--flat-table) — columns `Category`, `Amount`; rows ordered highest to lowest amount; subtitle shows current month via `as_of` / `month`.
