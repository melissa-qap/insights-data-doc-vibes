# Recurring spending

### Description

Lists all **active** recurring spending patterns across linked depository and credit accounts, detected from transaction history over the last 6 months. Each row shows the merchant, category, typical amount, detected frequency (weekly through annual, including semi-monthly), and last charge date.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier |
| `type` | Account type — must be `depository` or `credit` |

**Input:** `user_id = ?`. `type` in (`depository`, `credit`). Used to scope transactions to spending accounts.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow) |
| `date` | Posted date — used for interval detection and active filter |
| `pending` | Whether transaction has posted — must be false |
| `merchant_name` | Merchant display name — preferred for grouping |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Top-level category for display and spend filtering |

**Input:** `user_id = ?`. `account_id` in depository/credit account list. `date` within last 6 months. `pending = false`. Exclude `transaction_id` in `plaid_transactions_removed`.

#### `plaid_transactions_removed`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | ID of transaction removed from active set |

**Input:** `user_id = ?`. Used to build exclusion list for active transactions.

### Calculation / analysis

1. Set **detection window** to the last 6 months (`window_start` = today − 6 months, `window_end` = today).
2. Load depository and credit `account_id` values from `plaid_accounts` where `type` in (`depository`, `credit`).
3. Load `plaid_transactions` for those accounts in the window. Exclude pending and removed transactions.
4. **Spending eligibility** — exclude transactions where `personal_finance_category_primary` is `INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, or `LOAN_PAYMENTS`. Keep **outflows only** (`amount > 0`).
5. **Merchant key** — `display_merchant = COALESCE(merchant_name, name)`; normalize for grouping (trim whitespace, case-fold).
6. **Group candidates** by normalized `display_merchant` (across all scoped accounts).
7. Keep groups with **≥ 2** transactions.
8. **Amount consistency** — drop transactions where `amount` is outside **±20%** of the group median; re-check **≥ 2** transactions remain.
9. **Detect frequency** from the **median** gap in days between consecutive `date` values (sorted ascending). Assign the first matching bucket:
   - **Weekly:** 5–9 days
   - **Biweekly:** 10–15 days
   - **Semi-monthly:** 16–22 days
   - **Monthly:** 23–40 days
   - **Quarterly:** 75–105 days
   - **Annual:** 340–380 days
   - Groups outside these ranges are **excluded** (no stable cadence).
10. **Active filter** — keep only if `last_date` is within **1.5×** the bucket’s typical interval:
    - Weekly: ≤ 14 days ago
    - Biweekly: ≤ 23 days ago
    - Semi-monthly: ≤ 33 days ago
    - Monthly: ≤ 60 days ago
    - Quarterly: ≤ 158 days ago
    - Annual: ≤ 570 days ago
11. **Typical amount** — median of `amount` across occurrences in the group.
12. **Category** — `personal_finance_category_primary` from the most recent occurrence in the group; use `"UNCATEGORIZED"` when null.
13. **Build flat rows** — `{ merchant_name, category, frequency, typical_amount, last_date, occurrence_count }`. Use normalized `display_merchant` for `merchant_name` display.
14. **Sort rows** — by frequency order (weekly → biweekly → semi-monthly → monthly → quarterly → annual), then `typical_amount` descending.
15. **Optional rollup** — if showing a footer total (e.g. estimated monthly recurring spend), derive it from detail rows only (do not compute independently).

### Data output

| Field | Type | Description |
|---|---|---|
| `recurrences` | array | Flat list: `{ merchant_name, category, frequency, typical_amount, last_date, occurrence_count }` |
| `window_start` | date | Start of 6-month detection window |
| `window_end` | date | End of detection window |

### UI output

**Pattern:** [Flat table](../../ui-output-options.md#recurring-spending--flat-table) — columns `Merchant`, `Category`, `Frequency`, `Amount`, `Last date`.

### Notes

- **Not available in current Plaid schema:** Plaid Recurring Transactions / streams — recurrence is inferred from `plaid_transactions`; patterns may be missed or mislabeled.
- **Semi-monthly** vs **biweekly** are distinguished by median gap (10–15 vs 16–22 days).
- **Merchant name drift** (e.g. `NETFLIX.COM` vs `Netflix`) may split one subscription into multiple groups unless normalization is extended.
- The same merchant on multiple accounts is **one row** — transactions are grouped by merchant only, not per account.
- **Variable bills** (utilities) may fail the ±20% amount filter or fall outside frequency buckets.
- Credit card purchases on the card account are included; transfers and loan payments are excluded by category filter.
