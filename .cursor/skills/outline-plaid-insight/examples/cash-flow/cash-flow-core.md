# Cash flow core (shared partial)

### Description

Shared transaction table for cash flow insights — consumed by [monthly spending by category](monthly-spending-by-category.md), [cash inflow and outflow chart](cash-inflow-outflow-chart.md), [recurring transactions](recurring-transactions.md), [top 5 biggest purchases](top-5-biggest-purchases.md), and [late paycheck](../alerts/late-paycheck.md).

### Transaction table

Join `plaid_transactions` to `plaid_accounts` on `account_id`. Scope by `user_id`. Exclude `pending = true` and `removed = true`.

**Input:** `user_id = ?`

**Output:** `transactions[]`

**SQL pattern:**

```sql
SELECT t.*, a.type AS account_type
FROM plaid_transactions t
JOIN plaid_accounts a
  ON t.account_id = a.account_id
 AND t.user_id = a.user_id
WHERE t.user_id = ?
  AND t.pending = false
  AND t.removed = false
```

#### Columns

| Column | Description |
|---|---|
| `user_id` | User scope — filter all queries by this |
| `item_id` | Plaid Item that produced this row |
| `synced_at` | When this row was imported from Plaid |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Account the transaction belongs to |
| `account_type` | High-level account classification from `plaid_accounts.type` (`depository`, `credit`, `investment`, `loan`, `other`) — joined field |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date |
| `authorized_date` | Date the transaction was authorized — null when unavailable |
| `name` | Transaction description from the institution |
| `merchant_name` | Merchant display name — null when unavailable; preferred for grouping |
| `pending` | Whether the transaction has posted — always `false` in this table |
| `removed` | Whether the transaction was removed from the active set — always `false` in this table |
| `payment_channel` | How the transaction was initiated (`online`, `in store`, `other`) |
| `personal_finance_category_primary` | Top-level Plaid personal finance category — used for spend/income filtering and grouping |
| `personal_finance_category_detailed` | Detailed Plaid personal finance category |
| `personal_finance_category_confidence_level` | Plaid categorization confidence (`VERY_HIGH`, `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN`) |
| `location_city` | Transaction location city — null when unavailable |
| `location_region` | Transaction location region/state — null when unavailable |
| `iso_currency_code` | ISO-4217 currency code — null when unavailable |
| `unofficial_currency_code` | Unofficial currency code when ISO code is absent — null when unavailable |

All column names and types: [plaid-api-schema.md](../../plaid-api-schema.md). Callers own account scope, timeframe, category/amount filters, and label enrichment.
