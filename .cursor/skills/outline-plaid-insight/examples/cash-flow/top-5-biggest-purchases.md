# Top 5 biggest purchases

### Description

Shows the user's five largest individual spending transactions over the **trailing 30 days** across linked depository and credit accounts, so they can see their biggest one-off purchases at a glance.

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier |
| `name` | Account display name — used in output rows |

**Input:** `user_id = ?`. `name` used for account label join in step 9.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow) |
| `date` | Posted date — used in timeframe filter |
| `pending` | Must be false |
| `merchant_name` | Preferred display label |
| `name` | Fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Category for display and eligibility filter |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Window**
   - `window_end` = today's date
   - `window_start` = today − 30 days (inclusive range on `date`)
2. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id`
3. **Account scope**
   - Keep rows where `account_type` in (`depository`, `credit`)
4. **Filter by timeframe**
   - Keep rows where `date` is on or between `window_start` and `window_end` (inclusive)
5. **Apply eligibility**
   - Exclude `personal_finance_category_primary` in (`INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, `LOAN_PAYMENTS`)
   - Keep rows where `amount > 0`
6. **Enrich**
   - `display_label = COALESCE(merchant_name, name)`
   - `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
7. **Join account name**
   - Map `account_id` → `plaid_accounts.name` as `account_name`
8. **Rank**
   - Sort eligible transactions by `amount` descending
   - Tie-break: `date` descending, then `transaction_id` ascending
   - Take top 5
9. **Build rows**
   - `{ rank, transaction_id, merchant_name, category, amount, date, account_id, account_name }` for ranks 1–5
   - Use `display_label` for `merchant_name`
10. **Optional period total**
    - If surfacing total spend in the window, derive as sum of all eligible transactions (not just top 5)
    - Do not compute independently of the filtered transaction set
11. **Format output**
    - Apply [output formatting](../../SKILL.md#output-formatting): round all monetary fields (`amount`, `period_total_spend`) to 2 dp

### Data output

**Formatting:** Dollar fields — 2 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `purchases` | array | Top 5 transactions: `{ rank, transaction_id, merchant_name, category, amount, date, account_id, account_name }` — ordered by `rank` (1 = largest) |
| `window_start` | date | Start of trailing 30-day window |
| `window_end` | date | End of window (today) |
| `period_total_spend` | number \| optional | Optional rollup — sum of all eligible outflows in window |

### Notes

- Individual transactions, not merchant-level aggregation — a $800 Amazon order and a $200 Amazon order are two separate candidates.
- Investment buys (`plaid_investment_transactions`) are **out of scope**.
- Refunds (`amount < 0`) are excluded by eligibility filters; partial refunds on a purchase are not netted against the original transaction row.
- Credit card charges on the card account are included; paying the card from checking is excluded as `TRANSFER_OUT` / `TRANSFER_IN`.
