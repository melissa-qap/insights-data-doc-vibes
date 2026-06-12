# Recurring transactions

### Description

Lists all **active** recurring transaction patterns across linked accounts, detected from transaction history over the last **24 months** (required for annual and quarterly cadences). Detects both **outflow** charges (depository and credit) and **inflow** income (depository only). Each row shows the merchant or payer, account, Plaid category, product **group** (`bills`, `income`, or `subscriptions`), typical amount, detected frequency (weekly through annual, including semi-monthly), and last occurrence date. The `bills` group covers recurring rent, utilities, and loan payments (`RENT_AND_UTILITIES`, `LOAN_PAYMENTS`). The `income` group covers recurring inflows tagged `INCOME`. The `subscriptions` group covers all other recurring outflows, including uncategorized patterns.

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
| `account_id` | Account |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date — used in timeframe filter, interval detection, and active filter |
| `pending` | Whether transaction has posted — must be false |
| `merchant_name` | Merchant or payer display name — preferred for grouping |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Top-level category for display and group classification |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Windows**
   - `window_end` = today
   - **Display window** — last 6 months (metadata only; does not limit which transactions are loaded)
     - `window_start` = today − 6 months
2. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id`
3. **Filter by detection lookback**
   - `detection_lookback_start` = today − 24 months
   - Keep rows where `date` is on or between `detection_lookback_start` and `window_end` (inclusive)
4. **Split eligibility paths** — run steps 5–14 separately per path, then merge at step 15
   - **Shared exclusions:** exclude `personal_finance_category_primary` in (`TRANSFER_IN`, `TRANSFER_OUT`)
   - **Income path** — `personal_finance_category_primary = 'INCOME'` → `group = income` in output
     - Account scope: `account_type = 'depository'`
     - Keep rows where `amount < 0` (inflow)
     - Set `direction = 'inflow'`
   - **Outflow path** — all other eligible spending → `group = bills` or `subscriptions` in output (step 13)
     - Exclude `personal_finance_category_primary = 'INCOME'`
     - Account scope: `account_type` in (`depository`, `credit`)
     - Keep rows where `amount > 0`
     - Include `LOAN_PAYMENTS` and all other non-income outflow categories
     - Set `direction = 'outflow'`
5. **Enrich** (per path)
   - `display_label = COALESCE(merchant_name, name)`
   - `display_label_normalized` = trimmed, case-folded `display_label`
   - `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
6. **Group candidates**
   - By `display_label_normalized` + `account_id` + `direction`
7. **Minimum group size**
   - Keep groups with **≥ 2** transactions
8. **Amount consistency**
   - Use `ABS(amount)` for median and consistency checks (inflows are negative in source data)
   - Drop transactions where `ABS(amount)` is outside **±20%** of the group median
   - Re-check **≥ 2** transactions remain
9. **Detect frequency**
   - Compute from the **median** gap in days between consecutive `date` values (sorted ascending)
   - Assign the first matching bucket:
     - **Weekly:** 5–9 days
     - **Biweekly:** 10–15 days
     - **Semi-monthly:** 16–22 days
     - **Monthly:** 23–40 days
     - **Quarterly:** 75–105 days
     - **Annual:** 340–380 days
     - Groups outside these ranges are **excluded** (no stable cadence)
10. **Active filter**
    - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
      - Weekly: ≤ 14 days ago
      - Biweekly: ≤ 23 days ago
      - Semi-monthly: ≤ 33 days ago
      - Monthly: ≤ 60 days ago
      - Quarterly: ≤ 158 days ago
      - Annual: ≤ 570 days ago
11. **Typical amount**
    - Median of `ABS(amount)` across occurrences in the group (always positive in output)
12. **Category**
    - `category` from the most recent occurrence in the group
13. **Assign group** — evaluate in priority order (first match wins) using `category` from step 12
    - **`income`** — `category = 'INCOME'`
    - **`bills`** — `category` in (`RENT_AND_UTILITIES`, `LOAN_PAYMENTS`)
    - **`subscriptions`** — all remaining outflow recurrences, including `category = 'UNCATEGORIZED'`
14. **Account**
    - `account_id` from the group key
    - `account_name` = `plaid_accounts.name` for that `account_id` (latest snapshot)
15. **Merge paths** — combine outflow and inflow recurrence rows into one list
16. **Build flat rows and monthly rollups** — derive rollups from `recurrences[]` only (do not compute independently)
    - **Recurrence rows** — `{ merchant_name, account_id, account_name, category, group, direction, frequency, typical_amount, last_date, occurrence_count }`
      - Use `display_label_normalized` for `merchant_name` display
    - **Monthly equivalent** per recurrence: semi-monthly `× 2`, monthly `× 1`, quarterly `÷ 3`, annual `÷ 12`, weekly `× 4.33`, biweekly `× 2.17`
    - **By group** — one row per product group (`bills`, `income`, `subscriptions`): `{ group, estimated_monthly_total }`
      - `estimated_monthly_total` = sum of monthly equivalents for recurrences in the group
    - **By account** — one row per `account_id`: `{ account_id, account_name, estimated_monthly_total }`
      - `estimated_monthly_total` = sum of monthly equivalents across all recurrences on that account
17. **Sort rows**
    - By group order (`income` → `bills` → `subscriptions`), then frequency order (weekly → biweekly → semi-monthly → monthly → quarterly → annual), then `typical_amount` descending

### Data output

| Field | Type | Description |
|---|---|---|
| `recurrences` | array | Flat list: `{ merchant_name, account_id, account_name, category, group, direction, frequency, typical_amount, last_date, occurrence_count }` |
| `by_group` | array | Product group rollups: `{ group, estimated_monthly_total }` — derived from `recurrences[]` only |
| `by_account` | array | Account rollups: `{ account_id, account_name, estimated_monthly_total }` — derived from `recurrences[]` only |
| `detection_lookback_start` | date | Start of 24-month transaction lookback for pattern detection (today − 24 months) |
| `window_start` | date | Start of 6-month display window |
| `window_end` | date | End of detection and display window (today) |

### Notes

- **Detection lookback vs display window:** Transactions are loaded from `detection_lookback_start` through `window_end` (24 months). Annual patterns need ≥ 2 charges ~365 days apart, so annual patterns are missed when less than ~13 months of history falls in that window. `window_start` / `window_end` (6 months) are display-window metadata only — they do not filter the transaction set.
- **Not available in current Plaid schema:** Plaid Recurring Transactions / streams — recurrence is inferred from `plaid_transactions`; patterns may be missed or mislabeled.
- **Classification:** `personal_finance_category_primary = INCOME` → `group = income` (output `category` is `INCOME`). `RENT_AND_UTILITIES` or `LOAN_PAYMENTS` → `group = bills`. All other outflows → `group = subscriptions`.
- **Income path:** `personal_finance_category_primary = INCOME` on depository accounts with inflow (`amount < 0`); credit-account inflows are excluded.
- **Bills scope:** Primary-level categories include rent and utilities together under `RENT_AND_UTILITIES`; insurance, taxes, and other bill-like charges classify as `subscriptions` unless primary category changes.
- **Semi-monthly** vs **biweekly** are distinguished by median gap (10–15 vs 16–22 days).
- **Merchant name drift** (e.g. `NETFLIX.COM` vs `Netflix`) may split one subscription into multiple groups unless normalization is extended.
- The same merchant on **different accounts** produces **separate rows** — grouping key is merchant + `account_id` + `direction`.
- **Variable bills** (e.g. utilities with seasonal usage) may fail the ±20% amount filter or fall outside frequency buckets.
