# Recurring transactions (V2)

### Description

Lists all **active** recurring transaction patterns across linked accounts, detected from **all available** transaction history. Runs a single detection pipeline over every row in [cash flow core](cash-flow-core.md) (depository, credit, and student loan accounts supported by `plaid_transactions`). Each row shows the merchant or payer, account, Plaid category, product **group** (`bills`, `income`, `subscriptions`, `transfers`, or `other_inflow`), typical amount, detected frequency (weekly, biweekly, monthly, or annual), last occurrence date, projected next occurrence date, and an `occurrences[]` timeline of actual past charges/deposits (default last 3 calendar months) and projected future dates (default next 6 calendar months). The `bills` group covers recurring rent, utilities, loan payments, insurance, taxes, and bank fees (`RENT_AND_UTILITIES`, `LOAN_PAYMENTS`, `GENERAL_SERVICES`, `GOVERNMENT_AND_NON_PROFIT`, `BANK_FEES`). The `income` group covers recurring inflows tagged `INCOME`. The `subscriptions` group covers all other recurring outflows, including uncategorized patterns. The `transfers` group covers recurring `TRANSFER_IN` and `TRANSFER_OUT` patterns. The `other_inflow` group covers recurring inflows that are not income or transfers (e.g. refunds, credits).

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier |
| `name` | Account display name — used in output rows |
| `mask` | Optional last digits — used for account label (`name • mask`) |

**Input:** `user_id = ?`. Latest snapshot. `name` and `mask` used for account label join in step 9.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Account |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date — used in interval detection and active filter |
| `pending` | Whether transaction has posted — must be false |
| `merchant_name` | Merchant or payer display name — preferred for grouping |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Top-level category for display and group classification |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); no caller-side category, account-type, or amount filters.

**Parameters**

| Parameter | Type | Default | Notes / options |
|---|---|---|---|
| `history_months` | integer | `3` | Calendar months of actual occurrences to include (`history_start` through `as_of`). Min 0, max 12. Pass `0` to omit actual occurrences. |
| `projection_months` | integer | `6` | Calendar months of projected occurrences to include (`as_of` through `projection_end`). Min 0, max 12. Pass `0` to omit projected occurrences. |

### Calculation / analysis

1. **Reference windows**
   - `as_of` = today
   - `history_start` = `as_of` − `history_months` calendar months
   - `projection_end` = `as_of` + `projection_months` calendar months
   - `window_end` = `as_of` — upper bound for `next_date` projection
2. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id` (all stored history; no date lower bound)
3. **Clean up data** — prepare grouped transaction sets for recurrence detection (no category, account-type, or amount filters before grouping):
   - **Set direction** — `direction = 'inflow'` when `amount < 0`; `direction = 'outflow'` when `amount > 0`; drop rows where `amount = 0`
   - **Enrich** — `display_label = COALESCE(merchant_name, name)`; `display_label_normalized` = trimmed, case-folded `display_label`; `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
   - **Group candidates** — by `display_label_normalized` + `account_id` + `direction`
   - **Minimum group size** — keep groups with **≥ 2** transactions
   - **Amount consistency** — use `ABS(amount)` for median and consistency checks (inflows are negative in source data); drop transactions where `ABS(amount)` is outside **±20%** of the group median; re-check **≥ 2** transactions remain
4. **Detect frequency**
   - `median_gap_days` = median gap in days between consecutive `date` values (sorted ascending)
   - Assign the first matching bucket:
     - **Weekly:** 5–9 days
     - **Biweekly:** 10–22 days (includes former semi-monthly cadences)
     - **Monthly:** 23–40 days
     - **Annual:** 340–380 days
     - Groups outside these ranges are **excluded** (no stable cadence)
5. **Active filter**
    - `last_date` = most recent `date` in the group
    - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
      - Weekly: ≤ 14 days ago
      - Biweekly: ≤ 33 days ago
      - Monthly: ≤ 60 days ago
      - Annual: ≤ 570 days ago
6. **Typical amount**
    - Median of `ABS(amount)` across occurrences in the group (always positive in output)
7. **Category**
    - `category` from the most recent occurrence in the group
8. **Assign group** — evaluate in priority order (first match wins) using `category` from step 7 and `direction` from step 3
    - **`income`** — `direction = 'inflow'` AND `category = 'INCOME'`
    - **`transfers`** — `category` in (`TRANSFER_IN`, `TRANSFER_OUT`)
    - **`bills`** — `direction = 'outflow'` AND `category` in (`RENT_AND_UTILITIES`, `LOAN_PAYMENTS`, `GENERAL_SERVICES`, `GOVERNMENT_AND_NON_PROFIT`, `BANK_FEES`)
    - **`subscriptions`** — `direction = 'outflow'` AND not matched above
    - **`other_inflow`** — `direction = 'inflow'` AND not matched above
9. **Account**
    - `account_id` from the group key
    - `account_name` = `plaid_accounts.name` for that `account_id` (latest snapshot)
    - `account_mask` = `plaid_accounts.mask` for that `account_id` (nullable)
10. **Project next date**
    - Start at `last_date`; advance one period at a time until date ≥ `as_of`; first such date = `next_date`
    - **Weekly, biweekly:** add `median_gap_days` per step
    - **Monthly:** advance to same day-of-month as `last_date` in the next calendar month (clamp to month-end when needed, e.g. Jan 31 → Feb 28)
    - **Annual:** add 1 calendar year per step (same day-of-month clamp rule)
11. **Build occurrence timeline** (per recurrence; skip when both `history_months = 0` and `projection_months = 0`)
    - Use transactions that survived step 3 amount-consistency filter
    - **Actual occurrences** (when `history_months > 0`) — keep rows where `history_start <= date <= as_of`; emit `{ date, amount, kind: 'actual', transaction_id }` where `amount = ABS(amount)` and `transaction_id` from `plaid_transactions`
    - **Projected occurrences** (when `projection_months > 0`) — start at `last_date`; repeatedly advance one period (same rules as step 10) until date > `projection_end`; keep dates where `date >= as_of` AND `date <= projection_end`; emit `{ date, amount: typical_amount, kind: 'projected' }` (omit `transaction_id`)
    - Concatenate actual + projected; sort ascending by `date`; attach as `occurrences[]` on the recurrence row
    - When projections exist, `next_date` equals the first `kind: 'projected'` entry in `occurrences[]` on or after `as_of`
12. **Build flat rows and monthly rollups** — derive rollups from `recurrences[]` only (do not compute independently)
    - **Recurrence rows** — `{ recurrence_id, merchant_name, account_id, account_name, account_mask, category, group, direction, frequency, median_gap_days, typical_amount, last_date, next_date, occurrence_count, occurrences }`
      - `recurrence_id` = deterministic hash of `account_id` + `direction` + `display_label_normalized` (trim, case-folded merchant key from step 3)
      - Use `display_label_normalized` for `merchant_name` display
    - **Monthly equivalent** per recurrence: monthly `× 1`, annual `÷ 12`, weekly `× 4.33`, biweekly `× 2.17`
    - **By group** — one row per product group present in `recurrences[]` (`bills`, `income`, `subscriptions`, `transfers`, `other_inflow`): `{ group, estimated_monthly_total }`
      - `estimated_monthly_total` = sum of monthly equivalents for recurrences in the group
    - **By account** — one row per `account_id`: `{ account_id, account_name, account_mask, estimated_monthly_total }`
      - `estimated_monthly_total` = sum of monthly equivalents across all recurrences on that account
13. **Format output**
    - Apply [output formatting](../../SKILL.md#output-formatting): round all monetary fields (`typical_amount`, `estimated_monthly_total`, `occurrences[].amount`) to 2 dp; leave `median_gap_days` and `occurrence_count` as integers (no rounding)

### Data output

**Formatting:** Dollar fields — 2 dp ([SKILL.md](../../SKILL.md#output-formatting)); `median_gap_days` and `occurrence_count` excluded (counts).

| Field | Type | Description |
|---|---|---|
| `as_of` | date | Reference date for occurrence windows |
| `history_start` | date | Start of actual-occurrence window |
| `projection_end` | date | End of projected-occurrence window |
| `recurrences` | array | Flat list: `{ recurrence_id, merchant_name, account_id, account_name, account_mask, category, group, direction, frequency, median_gap_days, typical_amount, last_date, next_date, occurrence_count, occurrences }` — `recurrence_id` = hash of `account_id` + `direction` + normalized `merchant_name`; `group` is one of `bills`, `income`, `subscriptions`, `transfers`, `other_inflow`; `occurrences[]` is `{ date, amount, kind, transaction_id? }` sorted ascending (`kind` is `actual` or `projected`) |
| `by_group` | array | Product group rollups: `{ group, estimated_monthly_total }` — derived from `recurrences[]` only; includes all groups with active recurrences |
| `by_account` | array | Account rollups: `{ account_id, account_name, account_mask, estimated_monthly_total }` — derived from `recurrences[]` only |

### Notes

- **Transaction history:** All stored transactions from cash-flow-core are used for detection (no lookback cap). Annual patterns still require ≥ 2 occurrences at the expected cadence in whatever history exists — patterns may still be missed when total history spans less than ~13 months. Cadences between monthly and annual (e.g. quarterly) are not detected.
- **Plaid alternative:** [Recurring transactions (V1)](recurring-transactions-v1.md) uses Plaid-detected streams via `/transactions/recurring/get` when the Recurring Transactions add-on is available. V2 infers patterns from `plaid_transactions` when it is not.
- **Account scope:** All account types present in `plaid_transactions` (depository, credit, student loan). Investment transactions are not in this table — see [recurring investments](../investment-account/recurring-investments.md).
- **Classification:** `direction = 'inflow'` AND `category = 'INCOME'` → `group = income`. `category` in (`TRANSFER_IN`, `TRANSFER_OUT`) → `group = transfers`. `direction = 'outflow'` AND `RENT_AND_UTILITIES`, `LOAN_PAYMENTS`, `GENERAL_SERVICES`, `GOVERNMENT_AND_NON_PROFIT`, or `BANK_FEES` → `group = bills`. All other outflows → `group = subscriptions`. Remaining inflows → `group = other_inflow`.
- **Occurrence timeline:** Default windows are last 3 calendar months (actual) and next 6 calendar months (projected). Actual rows use posted transaction dates and amounts; projected rows use `typical_amount`. Use parent `direction` to interpret inflow vs outflow. Pass `history_months = 0` or `projection_months = 0` to omit one side.
- **Payload size:** Weekly recurrences may emit ~26 projected rows each over 6 months. Tune `projection_months` / `history_months` (max 12) for lighter responses.
- **Transfers:** Recurring transfers between owned accounts (e.g. monthly savings moves) may surface in `transfers`. Clients focused on bills, income, or subscriptions can filter by `group`.
- **Credit inflows:** Refunds or rewards on credit accounts may form recurrence groups — mostly `other_inflow`, or mis-tagged `income` when Plaid categorizes as `INCOME`.
- **Student loans:** Loan payment patterns on student-loan accounts may appear in `bills` via `LOAN_PAYMENTS`.
- **Bills scope:** Primary-level `GENERAL_SERVICES` is broad (insurance, tuition, childcare, automotive, and other services). Cloud storage or software tagged under `GENERAL_SERVICES` may classify as `bills`; a future refinement could use `personal_finance_category_detailed` to split bill-like services from subscriptions. `UNCATEGORIZED` recurring outflows (e.g. rent without a Plaid category) remain `subscriptions`.
- **Biweekly bucket:** Merges every-other-week (~14-day) and twice-per-month (~15–22-day) cadences; projection uses `median_gap_days` stepping.
- **Merchant name drift** (e.g. `NETFLIX.COM` vs `Netflix`) may split one subscription into multiple groups unless normalization is extended.
- The same merchant on **different accounts** produces **separate rows** — grouping key is merchant + `account_id` + `direction`.
- **Variable bills** (e.g. utilities with seasonal usage) may fail the ±20% amount filter or fall outside frequency buckets.
- **`next_date` projection:** Inferred from historical cadence, not institution-confirmed. If a charge posted after `last_date` but detection has not re-run, `next_date` may be stale until next sync. When `projection_months > 0`, `next_date` matches the first projected entry in `occurrences[]` on or after `as_of`.
- **Client presentation:** Account label — `{account_name} • {account_mask}` when `account_mask` is present; else `account_name` only. Relative status (e.g. "Due today", "Due tomorrow") — client compares `next_date` to today; omit rows where `next_date < today` from Upcoming views. No server `status` field.
- **Upcoming tab:** Filter `next_date >= today` or use projected entries in `occurrences[]` where `kind = 'projected'`. For calendar views beyond the default 6-month window, extend projection client-side using the same frequency-aware advance rules from step 10.
- **[Late paycheck](../alerts/late-paycheck.md)** can consume `next_date` and `median_gap_days` from `recurrences[]` where `group = 'income'` instead of recomputing `expected_next_date`.
