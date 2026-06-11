# Excess checking cash

### Description

Flags when the user's combined **checking** balance exceeds a recommended cushion: **115% of their average monthly spending** over the trailing 3 months (spend estimated from depository + credit transactions, same category filters as [monthly spending by category](../cash-flow/monthly-spending-by-category.md)).

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Checking account identifier |
| `name` | Display name |
| `mask` | Optional last digits for UI |
| `type` | Must be `depository` |
| `subtype` | Must be `checking` (case-insensitive match on `LOWER(subtype)`) |
| `balances_current` | Balance for excess comparison (per schema: use `current`, not `balances_available`) |
| `synced_at` | Latest snapshot timestamp |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX`). `type = depository` and `LOWER(subtype) = 'checking'`. Exclude null `balances_current`.

#### `plaid_accounts` (spend scope)

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Accounts for spend denominator |
| `type` | `depository` or `credit` |

**Input:** `user_id = ?`. Latest snapshot. `type` in (`depository`, `credit`). Used only to build the transaction `account_id` list (not for checking balance).

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique id |
| `account_id` | Spending account |
| `amount` | Outflow positive; refunds negative |
| `date` | Posted date for 3-month window |
| `pending` | Must be `false` |
| `personal_finance_category_primary` | Spend filtering |

**Input:** `user_id = ?`. `account_id` in depository/credit account list (same scope as [top 5 biggest purchases](../cash-flow/top-5-biggest-purchases.md)). `date` between `spend_window_start` and `spend_window_end` (inclusive). `pending = false`. `removed = false`.

### Calculation / analysis

1. **Checking snapshot**
   - Load latest `plaid_accounts` per schema current-state pattern
   - Filter to checking accounts
   - Build `checking_accounts[]` = `{ account_id, name, mask, balance }` where `balance = balances_current`
   - `total_checking_balance` = sum of `balance` (derive from detail rows only)
2. **Spend window**
   - `spend_window_end` = today
   - `spend_window_start` = today − 3 calendar months (same rolling style as [recurring spending](../cash-flow/recurring-spending.md) 6-month window)
3. **Spend scope**
   - Depository + credit `account_id` values from latest accounts where `type` in (`depository`, `credit`)
4. **Eligible spend**
   - Load transactions in window
   - Exclude pending and removed
   - Exclude categories `INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, `LOAN_PAYMENTS` (align with monthly spending)
   - Net per transaction into spend total (positive outflows + negative refunds)
5. **`total_spend_3mo`**
   - Sum of eligible spend in the window
6. **`monthly_avg_spend`**
   - `total_spend_3mo / 3` (fixed divisor = 3 months)
7. **`buffer_pct`**
   - `0.15`
8. **`recommended_minimum`**
   - `monthly_avg_spend * (1 + buffer_pct)` (= × 1.15)
   - Round only at presentation layer if needed
9. **`excess_cash`**
   - `max(0, total_checking_balance - recommended_minimum)`
10. **`has_excess`**
    - `excess_cash > 0`
11. **`as_of`**
    - `synced_at` from the checking snapshot (MAX across included checking rows)

**Rollup rule:** `total_checking_balance` must equal sum of `checking_accounts[].balance`.

### Data output

| Field | Type | Description |
|---|---|---|
| `has_excess` | boolean | True when `excess_cash > 0` |
| `excess_cash` | number | Dollars above recommended minimum (0 if none) |
| `total_checking_balance` | number | Sum of checking balances |
| `recommended_minimum` | number | `monthly_avg_spend * 1.15` |
| `monthly_avg_spend` | number | `total_spend_3mo / 3` |
| `buffer_pct` | number | `0.15` |
| `total_spend_3mo` | number | Eligible spend in window (audit/debug) |
| `spend_window_start` | date | Start of 3-month spend window |
| `spend_window_end` | date | End of spend window |
| `checking_accounts` | array | `{ account_id, name, mask, balance }` sorted by `balance` descending |
| `as_of` | timestamp | Latest checking sync used |

### UI output

**Pattern:** [Excess checking cash — insight card](../../ui-output-options.md#excess-checking-cash--insight-card)

### Notes

- **Savings excluded** from balance by design; product copy may suggest moving excess to savings (out of scope for Plaid tables).
- **`balances_available`:** Not used; many institutions omit it.
- **No spend history:** If `total_spend_3mo = 0`, `recommended_minimum = 0` and all checking balance counts as excess — hide the insight or show a neutral "insufficient spend history" state.
- **No checking accounts:** Omit insight or empty state.
- **Credit card spend included** in `monthly_avg_spend` so the buffer reflects real outflows; card payments from checking are excluded as `TRANSFER_OUT` (same as top 5 biggest purchases).
- **Not available in current Plaid schema:** User-defined target amounts — would need an extension preferences table.
