# Subscription price increase

### Description

Alerts when one or more **active recurring subscription-like charges** increase by **≥ 5%** compared to the median of prior charges at the same merchant. Subscription candidates come from [recurring transactions](../cash-flow/recurring-transactions.md) (`group = subscriptions`); per-occurrence transaction amounts are used for the price-change comparison (not `typical_amount`, which includes the latest charge).

### Required input data

#### Upstream: [recurring transactions](../cash-flow/recurring-transactions.md)

Run recurring transactions for `user_id = ?`. Use its output:

| Field | Description |
|---|---|
| `recurrences` | Active recurring patterns: `{ merchant_name, account_id, account_name, category, group, direction, frequency, typical_amount, last_date, occurrence_count }` |
| `detection_lookback_start` | Start of 24-month transaction lookback for pattern detection (today − 24 months) |
| `window_start` | Start of 6-month display window |
| `window_end` | End of detection and display window (today) |

**Input:** Full recurring transactions calculation (Plaid tables and filters documented there). This insight does not redefine recurrence detection.

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `account_id` | Spending account |
| `amount` | Transaction amount (positive = outflow) |
| `date` | Posted date — used for prior-median and latest charge |
| `pending` | Must be `false` |
| `merchant_name` | Merchant display name — preferred for matching |
| `name` | Transaction description — fallback when `merchant_name` is null |
| `personal_finance_category_primary` | Category filter (same exclusions as recurring transactions outflow path) |

**Input:** Same detection lookback (`detection_lookback_start`–`window_end`), account scope, and eligibility filters as [recurring transactions](../cash-flow/recurring-transactions.md) outflow path. Used only to load **per-occurrence amounts** for each subscription candidate — `typical_amount` from `recurrences[]` cannot substitute for `prior_median` / `latest_amount`.

### Calculation / analysis

1. **Recurring candidates** — Run [recurring transactions](../cash-flow/recurring-transactions.md) steps 1–10 (through active filter). Capture `recurrences[]`, `detection_lookback_start`, `window_start`, `window_end`.
2. **Subscription filter** — Keep rows where `group = 'subscriptions'`. Weekly and biweekly gym or streaming charges remain eligible when classified as subscriptions.
3. **Per-merchant occurrences** — For each candidate, load matching `plaid_transactions`: normalized `display_merchant = COALESCE(merchant_name, name)` equals recurrence `merchant_name` (trim, case-fold) **and** `account_id` equals recurrence `account_id`. Re-apply ±20% amount consistency against the group median (recurring transactions step 8). Require **≥ 2** occurrences.
4. **Prior median** — Sort occurrences by `date` ascending. Let `latest` = last occurrence. `prior_median` = median of `amount` for all occurrences **before** `latest.date`.
5. **Increase detection** — Flag when `prior_median > 0` and `latest.amount > prior_median * 1.05`.
6. **Derived fields per increase:**
   - `increase_amount` = `latest.amount - prior_median`
   - `increase_pct` = `increase_amount / prior_median`
   - `monthly_equivalent_increase` — normalize `increase_amount` by frequency: semi-monthly `× 2`, monthly `× 1`, quarterly `÷ 3`, annual `÷ 12`
7. **Build `increases[]`** — `{ merchant_name, category, frequency, prior_median, latest_amount, increase_amount, increase_pct, monthly_equivalent_increase, latest_date, occurrence_count }` from recurrence row + step 6 fields. `latest_date` = `latest.date`. Sort by `monthly_equivalent_increase` descending.
8. **Rollups (from detail rows only):**
   - `has_increases` = `increases.length > 0`
   - `increase_count` = `increases.length`
   - `estimated_monthly_impact` = sum of `monthly_equivalent_increase` across `increases[]`

### Data output

| Field | Type | Description |
|---|---|---|
| `has_increases` | boolean | True when any subscription passed the 5% threshold |
| `increase_count` | number | Count of increased subscriptions |
| `estimated_monthly_impact` | number | Sum of normalized monthly equivalents (derive from `increases[]` only) |
| `increases` | array | `{ merchant_name, category, frequency, prior_median, latest_amount, increase_amount, increase_pct, monthly_equivalent_increase, latest_date, occurrence_count }` |
| `detection_lookback_start` | date | From recurring transactions output |
| `window_start` | date | From recurring transactions output (6-month display window) |
| `window_end` | date | From recurring transactions output |

### Notes

- **Depends on [recurring transactions](../cash-flow/recurring-transactions.md)** for candidate detection; share implementation or cache `recurrences[]` rather than duplicating detection steps.
- **Do not use `typical_amount`** for increase detection — it is the median of all occurrences including the latest charge.
- **Not available in current Plaid schema:** Plaid Recurring Transactions / streams — recurrence and subscription identity are inferred; patterns may be missed or mislabeled (same caveats as recurring transactions).
- **Merchant name drift** may split one subscription into multiple groups unless normalization is extended.
- **Tax, promo, or prorated charges** can cause false positives despite the 5% threshold.
- **No user-defined subscription list** — would need an extension preferences table.
