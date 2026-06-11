# Late paycheck

### Description

Alerts when a **detected recurring paycheck** (stable `INCOME` inflow on depository accounts) is **≥ 3 calendar days past** its predicted next pay date with no matching deposit since the last occurrence. Each employer/payer source is evaluated independently.

Uses [cash flow core](../cash-flow/cash-flow-core.md) transaction table.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Depository account identifier |
| `name` | Display name — used for last-deposit account label |
| `mask` | Optional last digits for UI |
| `type` | Must be `depository` |

**Input:** `user_id = ?`. Latest snapshot. Used for `last_account_id` display and account labels.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Depository account receiving the deposit |
| `amount` | Negative = inflow; use `ABS(amount)` for consistency checks |
| `date` | Posted date — used in timeframe filter, cadence detection, and lateness |
| `pending` | Must be `false` |
| `merchant_name` | Payer display name — preferred for grouping |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Must be `INCOME` for paycheck candidates |

**Input:** `user_id = ?`. Loaded via [cash flow core](../cash-flow/cash-flow-core.md); filtered in caller steps.

### Calculation / analysis

1. **Detection window**
   - `window_start` = today − 6 months
   - `window_end` = today
2. **Load transaction table** — [cash flow core](../cash-flow/cash-flow-core.md) for `user_id`
3. **Account scope**
   - Keep rows where `account_type = 'depository'`
4. **Filter by timeframe**
   - Keep rows where `date` is on or between `window_start` and `window_end` (inclusive)
5. **Apply eligibility**
   - Exclude `personal_finance_category_primary` in (`TRANSFER_IN`, `TRANSFER_OUT`, `LOAN_PAYMENTS`)
   - Keep rows where `amount < 0`
   - Further filter: keep `personal_finance_category_primary = 'INCOME'` only
6. **Enrich**
   - `display_label = COALESCE(merchant_name, name)`
   - `display_label_normalized` = trimmed, case-folded `display_label`
7. **Group by payer**
   - By `display_label_normalized` across all depository accounts (one paycheck source per employer/payer)
8. **Minimum history**
   - Keep groups with **≥ 3** occurrences
9. **Amount consistency**
   - Drop occurrences where `ABS(amount)` is outside **±20%** of the group median
   - Re-check **≥ 3** occurrences remain
10. **Detect frequency**
    - Compute median gap in days between consecutive `date` values (sorted ascending)
    - Assign the first matching bucket (same ranges as [recurring spending](../cash-flow/recurring-spending.md)):
      - **Biweekly:** 10–15 days
      - **Semi-monthly:** 16–22 days
      - **Monthly:** 23–40 days
    - Exclude groups outside these buckets or matching weekly, quarterly, or annual ranges
11. **Active filter**
    - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
      - Biweekly: ≤ 23 days ago
      - Semi-monthly: ≤ 33 days ago
      - Monthly: ≤ 60 days ago
12. **Typical amount**
    - Median of `ABS(amount)` across occurrences in the group
13. **Last occurrence metadata**
    - `last_date` = most recent `date` in the group
    - `last_account_id` = `account_id` of the most recent occurrence
    - `occurrence_count` = count of occurrences after filters
14. **Expected next pay date**
    - `median_gap_days` = median gap from step 10
    - `expected_next_date` = `last_date + median_gap_days`
15. **Grace and lateness**
    - `grace_days` = `3`
    - **New deposit check:** matching `display_label_normalized`, `amount < 0`, `INCOME`, `date > last_date`, and `ABS(amount)` within ±20% of `typical_amount`
    - `is_late` = `(today >= expected_next_date + grace_days)` AND no new deposit found
    - `days_late` = `today - expected_next_date` when `is_late`; otherwise `0`
16. **Build `late_paychecks[]`**
    - One object per late source: `{ payer_name, typical_amount, frequency, last_date, expected_next_date, days_late, last_account_id, occurrence_count }`
    - `payer_name` = `display_label_normalized`
    - Sort by `days_late` descending
17. **Rollups (from detail rows only):**
    - `has_late_paychecks` = `late_paychecks.length > 0`
    - `late_count` = `late_paychecks.length`

### Data output

| Field | Type | Description |
|---|---|---|
| `has_late_paychecks` | boolean | True when any paycheck source is late |
| `late_count` | number | Count of late paycheck sources |
| `late_paychecks` | array | `{ payer_name, typical_amount, frequency, last_date, expected_next_date, days_late, last_account_id, occurrence_count }` sorted by `days_late` descending |
| `grace_days` | number | `3` |
| `window_start` | date | Start of 6-month detection window |
| `window_end` | date | End of detection window |

### UI output

**Pattern:** [Late paycheck — insight card](../../ui-output-options.md#late-paycheck--insight-card)

### Notes

- **Cadence inferred** from transaction history — same limitations as [recurring spending](../cash-flow/recurring-spending.md) (missed merges, merchant name drift, variable pay).
- **Not available in current Plaid schema:** Plaid Recurring Transactions / payroll flags; user-defined pay schedule or employer list — would need extension preferences table.
- **TRANSFER_IN payroll** (P2P or inter-account) may be missed when categorized as transfer rather than `INCOME`.
- **Variable pay / bonuses** may fail the ±20% filter or split one employer into multiple payer groups.
- **New job with < 3 deposits:** Excluded by minimum history to reduce false positives.
- **No active paycheck patterns or none late:** Hide insight (no empty state).
