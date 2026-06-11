# Cash flow core (shared partial)

### Description

Shared transaction table for cash flow insights — consumed by [monthly spending by category](monthly-spending-by-category.md), [cash inflow and outflow chart](cash-inflow-outflow-chart.md), [recurring spending](recurring-spending.md), [top 5 biggest purchases](top-5-biggest-purchases.md), and [late paycheck](../alerts/late-paycheck.md).

### Transaction table

Join `plaid_transactions` to `plaid_accounts` on `account_id`. Scope by `user_id`. Exclude `pending = true` and `removed = true`.

Each row includes all `plaid_transactions` columns ([plaid-api-schema.md](../../plaid-api-schema.md)) plus:

| Column | Source |
|---|---|
| `account_type` | `plaid_accounts.type` |

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

Callers own account scope, timeframe, category/amount filters, and label enrichment.
