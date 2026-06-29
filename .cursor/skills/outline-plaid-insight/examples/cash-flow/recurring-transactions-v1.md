# Recurring transactions (V1)

### Description

Lists all **active** recurring transaction patterns from Plaid-detected streams imported via `/transactions/recurring/get` into `plaid_recurring_streams`. Maps into the **same product output** as [recurring transactions (V2)](recurring-transactions.md) for drop-in compatibility. Each row shows the merchant or payer, account, Plaid category, product **group** (`bills`, `income`, `subscriptions`, `transfers`, or `other_inflow`), typical amount, detected frequency (weekly, biweekly, semi_monthly, monthly, or annual), last occurrence date, projected next occurrence date, and an `occurrences[]` timeline of actual past charges/deposits (default last 3 calendar months) and projected future dates (default next 6 calendar months).

Uses `plaid_recurring_streams` and `plaid_recurring_stream_transactions` — not inferred from raw transaction history.

### Required input data

#### `plaid_recurring_streams`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `stream_id` | Plaid stream identifier — used as `recurrence_id` in output |
| `direction` | `inflow` or `outflow` |
| `account_id` | Account |
| `description` | Stream description — fallback display label |
| `merchant_name` | Merchant name when available |
| `first_date` | Earliest transaction in stream |
| `last_date` | Latest transaction in stream |
| `predicted_next_date` | Plaid-predicted next date (nullable) |
| `frequency` | Plaid cadence — see [enum_recurring_stream_frequency](../../plaid-api-schema.md#enum_recurring_stream_frequency) |
| `average_amount` | Signed average amount (use `ABS` in output) |
| `is_active` | Whether stream is still live |
| `status` | Stream lifecycle — see [enum_recurring_stream_status](../../plaid-api-schema.md#enum_recurring_stream_status) |
| `personal_finance_category_primary` | Top-level category for display and group classification |

**Input:** `user_id = ?`. Latest snapshot (`MAX(synced_at)`). Keep `is_active = true` and `status != 'TOMBSTONED'`. Exclude streams where `frequency = 'UNKNOWN'`.

#### `plaid_recurring_stream_transactions`

| Column | Description |
|---|---|
| `stream_id` | Parent stream |
| `transaction_id` | Member transaction |
| `sequence` | Sort order within stream |

**Input:** Rows for streams in the current snapshot; join on `stream_id` + matching `synced_at` / `user_id` / `item_id`.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `transaction_id` | Unique transaction identifier |
| `account_id` | Account |
| `amount` | Transaction amount (positive = outflow, negative = inflow) |
| `date` | Posted date — used for `occurrences[]` and `median_gap_days` |
| `pending` | Whether transaction has posted — must be false |

**Input:** Join via `plaid_recurring_stream_transactions.transaction_id`. Exclude `pending = true` and `removed = true`.

#### `plaid_accounts`

| Column | Description |
|---|---|
| `account_id` | Account identifier |
| `name` | Account display name — used in output rows |
| `mask` | Optional last digits — used for account label (`name • mask`) |

**Input:** `user_id = ?`. Latest snapshot. `name` and `mask` used for account label join in step 7.

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
2. **Load streams** — latest `plaid_recurring_streams` snapshot for `user_id`; keep `is_active = true`, `status != 'TOMBSTONED'`, `frequency != 'UNKNOWN'`
3. **Set direction** — from `direction` column (`inflow` / `outflow`)
4. **Display label** — `display_label = COALESCE(merchant_name, description)`; use for `merchant_name` in output
5. **Category** — `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
6. **Assign group** — evaluate in priority order (first match wins) using `category` from step 5 and `direction` from step 3 (same rules as [V2 step 8](recurring-transactions.md)):
   - **`income`** — `direction = 'inflow'` AND `category = 'INCOME'`
   - **`transfers`** — `category` in (`TRANSFER_IN`, `TRANSFER_OUT`)
   - **`bills`** — `direction = 'outflow'` AND `category` in (`RENT_AND_UTILITIES`, `LOAN_PAYMENTS`, `GENERAL_SERVICES`, `GOVERNMENT_AND_NON_PROFIT`, `BANK_FEES`)
   - **`subscriptions`** — `direction = 'outflow'` AND not matched above
   - **`other_inflow`** — `direction = 'inflow'` AND not matched above
7. **Account**
   - `account_id` from stream
   - `account_name` = `plaid_accounts.name` for that `account_id` (latest snapshot)
   - `account_mask` = `plaid_accounts.mask` for that `account_id` (nullable)
8. **Map frequency** — Plaid enum → output lowercase:
   - `WEEKLY` → `weekly`
   - `BIWEEKLY` → `biweekly`
   - `SEMI_MONTHLY` → `semi_monthly`
   - `MONTHLY` → `monthly`
   - `ANNUALLY` → `annual`
9. **Typical amount** — `ABS(average_amount)` (always positive in output)
10. **Dates** — `last_date` from stream; `next_date` = `predicted_next_date` (nullable)
11. **`median_gap_days`** — when ≥2 linked `plaid_transactions` exist for the stream, compute median gap in days between consecutive `date` values (sorted ascending); else `null`
12. **`occurrence_count`** — count of `transaction_id` rows in `plaid_recurring_stream_transactions` for the stream
13. **Build occurrence timeline** (per stream; skip when both `history_months = 0` and `projection_months = 0`)
    - **Actual occurrences** (when `history_months > 0`) — join `plaid_recurring_stream_transactions` → `plaid_transactions`; keep rows where `history_start <= date <= as_of`, `pending = false`, `removed = false`; emit `{ date, amount: ABS(amount), kind: 'actual', transaction_id }`
    - **Projected occurrences** (when `projection_months > 0` and `predicted_next_date` is set) — start at `predicted_next_date`; repeatedly advance one period using Plaid frequency rules through `projection_end`:
      - **Weekly:** add 7 days
      - **Biweekly:** add 14 days
      - **Semi_monthly:** advance to same day-of-month in the next half-month window (15th or last day of month, matching Plaid semi-monthly cadence); when ambiguous, add 15 days
      - **Monthly:** advance to same day-of-month in next calendar month (clamp to month-end)
      - **Annual:** add 1 calendar year (same day-of-month clamp)
      - Keep dates where `date >= as_of` AND `date <= projection_end`; emit `{ date, amount: typical_amount, kind: 'projected' }` (omit `transaction_id`)
    - Concatenate actual + projected; sort ascending by `date`; attach as `occurrences[]`
    - When projections exist, `next_date` equals the first `kind: 'projected'` entry in `occurrences[]` on or after `as_of`; when no projections, use `predicted_next_date` from step 10
14. **Build flat rows and monthly rollups** — derive rollups from `recurrences[]` only (do not compute independently)
    - **Recurrence rows** — `{ recurrence_id, merchant_name, account_id, account_name, account_mask, category, group, direction, frequency, median_gap_days, typical_amount, last_date, next_date, occurrence_count, occurrences }`
      - `recurrence_id` = `stream_id`
      - `merchant_name` = `display_label` from step 4
    - **Monthly equivalent** per recurrence: monthly `× 1`, annual `÷ 12`, weekly `× 4.33`, biweekly `× 2.17`, semi_monthly `× 2`
    - **By group** — one row per product group present in `recurrences[]` (`bills`, `income`, `subscriptions`, `transfers`, `other_inflow`): `{ group, estimated_monthly_total }`
    - **By account** — one row per `account_id`: `{ account_id, account_name, account_mask, estimated_monthly_total }`
15. **Format output**
    - Apply [output formatting](../../SKILL.md#output-formatting): round all monetary fields (`typical_amount`, `estimated_monthly_total`, `occurrences[].amount`) to 2 dp; leave `median_gap_days` and `occurrence_count` as integers when present (no rounding)

### Data output

**Formatting:** Dollar fields — 2 dp ([SKILL.md](../../SKILL.md#output-formatting)); `median_gap_days` and `occurrence_count` excluded (counts).

| Field | Type | Description |
|---|---|---|
| `as_of` | date | Reference date for occurrence windows |
| `history_start` | date | Start of actual-occurrence window |
| `projection_end` | date | End of projected-occurrence window |
| `recurrences` | array | Flat list: `{ recurrence_id, merchant_name, account_id, account_name, account_mask, category, group, direction, frequency, median_gap_days, typical_amount, last_date, next_date, occurrence_count, occurrences }` — `recurrence_id` = `stream_id`; `group` is one of `bills`, `income`, `subscriptions`, `transfers`, `other_inflow`; `frequency` may include `semi_monthly`; `median_gap_days` nullable; `occurrences[]` is `{ date, amount, kind, transaction_id? }` sorted ascending (`kind` is `actual` or `projected`) |
| `by_group` | array | Product group rollups: `{ group, estimated_monthly_total }` — derived from `recurrences[]` only; includes all groups with active recurrences |
| `by_account` | array | Account rollups: `{ account_id, account_name, account_mask, estimated_monthly_total }` — derived from `recurrences[]` only |

### Notes

- **Source:** Plaid Recurring Transactions add-on — requires import via `/transactions/recurring/get` into `plaid_recurring_streams` and `plaid_recurring_stream_transactions`. See [plaid-api-schema.md](../../plaid-api-schema.md#transactionsrecurringget).
- **vs V2:** [Recurring transactions (V2)](recurring-transactions.md) infers patterns from all `plaid_transactions` via [cash flow core](cash-flow-core.md). V1 trusts Plaid's stream model (`status`, `EARLY_DETECTION`, etc.). Use V2 when the Recurring Transactions product is unavailable.
- **Account scope:** Depository and credit card accounts only (per Plaid). Student-loan and broader account coverage available in V2 only.
- **`recurrence_id`:** `stream_id` in V1 vs deterministic hash in V2 — clients switching versions must remap IDs.
- **`semi_monthly`:** Present in V1 (Plaid `SEMI_MONTHLY`); V2 merges semi-monthly cadences into `biweekly`.
- **`median_gap_days`:** Derived from stored transactions when ≥2 exist; `null` for early-detection streams with insufficient history.
- **Projection:** Starts from Plaid `predicted_next_date` using Plaid frequency advance rules; less flexible than V2's `median_gap_days` stepping.
- **Stream status:** `EARLY_DETECTION` streams are included when `is_active = true`. `TOMBSTONED` streams are excluded.
- **Classification:** Same PFC + direction rules as V2 — see [V2 Notes](recurring-transactions.md).
- **Occurrence timeline:** Default windows are last 3 calendar months (actual) and next 6 calendar months (projected). Actual rows use posted transaction dates and amounts from `plaid_transactions`; projected rows use `typical_amount`. Pass `history_months = 0` or `projection_months = 0` to omit one side.
- **Client presentation:** Account label — `{account_name} • {account_mask}` when `account_mask` is present; else `account_name` only. Relative status — client compares `next_date` to today; omit rows where `next_date < today` from Upcoming views.
- **Downstream alerts:** [Late paycheck](../alerts/late-paycheck.md) and [subscription price increase](../alerts/subscription-price-increase.md) may consume V1 or V2 `recurrences[]` — same output shape; filter by `group` as documented in each alert.
