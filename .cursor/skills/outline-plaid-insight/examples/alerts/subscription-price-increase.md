# Subscription price increase

### Description

Alerts when one or more **active recurring subscription-like charges** increase by **≥ 5%** compared to the median of prior charges at the same merchant. Subscription candidates come from [recurring transactions](../cash-flow/recurring-transactions.md) (`group = subscriptions`); per-occurrence transaction amounts are used for the price-change comparison (not `typical_amount`, which includes the latest charge).

### Required input data

#### Upstream: [recurring transactions](../cash-flow/recurring-transactions.md)

Run recurring transactions for `user_id = ?`. Use its output:

| Field | Description |
|---|---|
| `recurrences` | Active recurring patterns: `{ merchant_name, account_id, account_name, account_mask, category, group, direction, frequency, median_gap_days, typical_amount, last_date, next_date, occurrence_count }` |

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
| `personal_finance_category_primary` | Category — used for display in output rows |

**Input:** Same transaction scope as [recurring transactions](../cash-flow/recurring-transactions.md) (full cash-flow-core table, all available history). Used only to load **per-occurrence amounts** for each subscription candidate — `typical_amount` from `recurrences[]` cannot substitute for `prior_median` / `latest_amount`.

### Calculation / analysis

1. **Recurring candidates** — Run [recurring transactions](../cash-flow/recurring-transactions.md) steps 1–5 (through active filter). Capture `recurrences[]` for per-occurrence loading in step 3.
2. **Subscription filter** — Keep rows where `group = 'subscriptions'`. Weekly and biweekly gym or streaming charges remain eligible when classified as subscriptions.
3. **Per-merchant occurrences** — For each candidate, load matching `plaid_transactions` from all available history: normalized `display_merchant = COALESCE(merchant_name, name)` equals recurrence `merchant_name` (trim, case-fold) **and** `account_id` equals recurrence `account_id`. Re-apply ±20% amount consistency against the group median (recurring transactions step 3). Require **≥ 2** occurrences.
4. **Prior median** — Sort occurrences by `date` ascending. Let `latest` = last occurrence. `prior_median` = median of `amount` for all occurrences **before** `latest.date`.
5. **Increase detection** — Flag when `prior_median > 0` and `latest.amount > prior_median * 1.05`.
6. **Derived fields per increase:**
   - `increase_amount` = `latest.amount - prior_median`
   - `increase_pct` = `increase_amount / prior_median`
   - `monthly_equivalent_increase` — normalize `increase_amount` by frequency: biweekly `× 2.17`, monthly `× 1`, annual `÷ 12`, weekly `× 4.33`
7. **Build `increases[]`** — `{ merchant_name, category, frequency, prior_median, latest_amount, increase_amount, increase_pct, monthly_equivalent_increase, latest_date, occurrence_count }` from recurrence row + step 6 fields. `latest_date` = `latest.date`. Sort by `monthly_equivalent_increase` descending.
8. **Rollups (from detail rows only):**
   - `has_increases` = `increases.length > 0`
   - `increase_count` = `increases.length`
   - `estimated_monthly_impact` = sum of `monthly_equivalent_increase` across `increases[]`
9. **Format output**
   - Apply [output formatting](../../SKILL.md#output-formatting): dollar fields (`prior_median`, `latest_amount`, `increase_amount`, `monthly_equivalent_increase`, `estimated_monthly_impact`) to 2 dp; fraction percent fields (`increase_pct`) to 3 dp

### Data output

**Formatting:** Dollar fields — 2 dp; fraction percent fields — 3 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `has_increases` | boolean | True when any subscription passed the 5% threshold |
| `increase_count` | number | Count of increased subscriptions |
| `estimated_monthly_impact` | number | Sum of normalized monthly equivalents (derive from `increases[]` only) |
| `increases` | array | `{ merchant_name, category, frequency, prior_median, latest_amount, increase_amount, increase_pct, monthly_equivalent_increase, latest_date, occurrence_count }` |
| `increases[].increase_pct` | number | Fraction (e.g. `0.050` = 5%) — 3 dp |

### Notes

- **Depends on [recurring transactions](../cash-flow/recurring-transactions.md)** for candidate detection; share implementation or cache `recurrences[]` rather than duplicating detection steps. Ignore `occurrences[]` on recurrence rows — this alert uses summary fields only.
- **Do not use `typical_amount`** for increase detection — it is the median of all occurrences including the latest charge.
- **Not available in current Plaid schema:** Plaid Recurring Transactions / streams — recurrence and subscription identity are inferred; patterns may be missed or mislabeled (same caveats as recurring transactions).
- **Merchant name drift** may split one subscription into multiple groups unless normalization is extended.
- **Tax, promo, or prorated charges** can cause false positives despite the 5% threshold.
- **No user-defined subscription list** — would need an extension preferences table.
