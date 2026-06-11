# Recurring spending

### Description

Lists all **active** recurring spending patterns across linked depository and credit accounts, detected from transaction history over the last 6 months. Each row shows the merchant, spending account, category, typical amount, detected frequency (weekly through annual, including semi-monthly), and last charge date.

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier |
| `name` | Account display name — used in output rows |

**Input:** `user_id = ?`. `name` used for account label join in step 14.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow) |
| `date` | Posted date — used in timeframe filter, interval detection, and active filter |
| `pending` | Whether transaction has posted — must be false |
| `merchant_name` | Merchant display name — preferred for grouping |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Top-level category for display and eligibility filter |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Detection window**
   - Last 6 months
   - `window_start` = today − 6 months
   - `window_end` = today
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
   - `display_label_normalized` = trimmed, case-folded `display_label`
   - `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
7. **Group candidates**
   - By `display_label_normalized` + `account_id`
8. **Minimum group size**
   - Keep groups with **≥ 2** transactions
9. **Amount consistency**
   - Drop transactions where `amount` is outside **±20%** of the group median
   - Re-check **≥ 2** transactions remain
10. **Detect frequency**
    - Compute from the **median** gap in days between consecutive `date` values (sorted ascending)
    - Assign the first matching bucket:
      - **Weekly:** 5–9 days
      - **Biweekly:** 10–15 days
      - **Semi-monthly:** 16–22 days
      - **Monthly:** 23–40 days
      - **Quarterly:** 75–105 days
      - **Annual:** 340–380 days
      - Groups outside these ranges are **excluded** (no stable cadence)
11. **Active filter**
    - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
      - Weekly: ≤ 14 days ago
      - Biweekly: ≤ 23 days ago
      - Semi-monthly: ≤ 33 days ago
      - Monthly: ≤ 60 days ago
      - Quarterly: ≤ 158 days ago
      - Annual: ≤ 570 days ago
12. **Typical amount**
    - Median of `amount` across occurrences in the group
13. **Category**
    - `category` from the most recent occurrence in the group
14. **Account**
    - `account_id` from the group key
    - `account_name` = `plaid_accounts.name` for that `account_id` (latest snapshot)
15. **Build flat rows**
    - `{ merchant_name, account_id, account_name, category, frequency, typical_amount, last_date, occurrence_count }`
    - Use `display_label_normalized` for `merchant_name` display
16. **Sort rows**
    - By frequency order (weekly → biweekly → semi-monthly → monthly → quarterly → annual), then `typical_amount` descending
17. **Optional rollup**
    - If showing a footer total (e.g. estimated monthly recurring spend), derive it from detail rows only (do not compute independently)

### Data output

| Field | Type | Description |
|---|---|---|
| `recurrences` | array | Flat list: `{ merchant_name, account_id, account_name, category, frequency, typical_amount, last_date, occurrence_count }` |
| `window_start` | date | Start of 6-month detection window |
| `window_end` | date | End of detection window |

### UI output

**Pattern:** [Flat table](../../ui-output-options.md#recurring-spending--flat-table) — columns `Merchant`, `Account`, `Category`, `Frequency`, `Amount`, `Last date`.

### Notes

- **Not available in current Plaid schema:** Plaid Recurring Transactions / streams — recurrence is inferred from `plaid_transactions`; patterns may be missed or mislabeled.
- **Semi-monthly** vs **biweekly** are distinguished by median gap (10–15 vs 16–22 days).
- **Merchant name drift** (e.g. `NETFLIX.COM` vs `Netflix`) may split one subscription into multiple groups unless normalization is extended.
- The same merchant on **different accounts** produces **separate rows** — grouping key is merchant + `account_id`.
- **Variable bills** (utilities) may fail the ±20% amount filter or fall outside frequency buckets.
- Credit card purchases on the card account are included; transfers and loan payments are excluded by eligibility filters.
