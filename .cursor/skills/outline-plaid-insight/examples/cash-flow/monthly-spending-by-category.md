# Monthly spending by category

### Description

Shows how much the user spent in a **selected calendar month**, broken down by Plaid personal finance category, so they can see which categories drive the most outflows that month. Returns the **top 5** categories by spend for that month; all remaining categories roll into a single **`OTHER`** bucket. The caller passes `month`; the response includes ranked `categories[]` for that month plus `months[]` picker options (`YYYY-MM` strings, up to 12).

Uses [cash flow core](cash-flow-core.md) transaction table.

### Required input data

#### `plaid_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `transaction_id` | Unique transaction identifier |
| `amount` | Transaction amount (positive = outflow, negative = inflow/refund) |
| `date` | Posted date — used for selected-month filter and calendar month assignment |
| `pending` | Whether transaction has posted — must be false |
| `personal_finance_category_primary` | Top-level spend category for grouping |

**Input:** `user_id = ?`. Loaded via [cash flow core](cash-flow-core.md); filtered in caller steps.

**Parameters:**

| Parameter | Type | Default | Notes / options |
|---|---|---|---|
| `month` | string (`YYYY-MM`) | Last month in `months[]` | Calendar month to display; must appear in `months[]` — reject or error if out of range |

### Calculation / analysis

1. **Load transaction table** — [cash flow core](cash-flow-core.md) for `user_id`
2. **Month window**
   - `window_end` = calendar date of `MAX(date)` on loaded table (or lightweight pre-query)
   - `lookback_start` = first day of the calendar month that is 11 months before `window_end`'s month (12 calendar months inclusive, e.g. Jun 2025–May 2026 when latest transaction is in May 2026)
3. **Account scope**
   - Keep rows where `account_type` in (`depository`, `credit`)
4. **Apply eligibility**
   - Exclude `personal_finance_category_primary` in (`INCOME`, `TRANSFER_IN`, `TRANSFER_OUT`, `LOAN_PAYMENTS`)
   - No amount sign filter (refunds net into category totals)
5. **Floor window start**
   - `earliest_month` = `YYYY-MM` of `MIN(date)` on scoped, eligible rows
   - `window_start` = **later of** `lookback_start` and first calendar day of `earliest_month` (do not extend before first in-scope data)
6. **Build `months[]`**
   - Loop every calendar month from `window_start`'s month through `window_end`'s month (inclusive), including zero-activity months
   - Emit one `YYYY-MM` string per month
   - Sort ascending; cap at 12 items
7. **Resolve selected month**
   - `selected_month = month` param or default (`YYYY-MM` of `window_end`)
   - Validate `selected_month` appears in `months[]`; reject or error if out of range
8. **Filter to selected month**
   - `start_date` = first calendar day of `selected_month`
   - `end_date` = last calendar day of `selected_month`; when `selected_month` is the same calendar month as `window_end`, use `window_end` (partial latest month)
   - Keep scoped, eligible rows where `date` is on or between `start_date` and `end_date` (inclusive)
9. **Enrich**
   - `category = personal_finance_category_primary` or `"UNCATEGORIZED"` when null
10. **Sum by category**
    - Group filtered rows by `category`
    - For each group, `amount` = sum of transaction `amount` values
    - Plaid sign: positive = money out, negative = refund/credit; both count (refunds reduce the category total)
11. **Build full ranked list and `total_spend`**
    - One entry per category with activity: `{ category, amount }`
    - Sparse — omit categories with no transactions
    - `total_spend` = sum of all category `amount` values (full uncapped list — not limited to top 5)
    - Compute `percent` = `amount / total_spend` when `total_spend > 0`, else `0`
    - When `total_spend <= 0` (refunds exceed outflows), set all `percent` values to `0`
    - Negative category `amount` values are allowed; `percent` can be negative when `total_spend > 0`
    - Sort by `amount` descending; tie-break: `category` ascending
    - Set top-level `month`, `start_date`, `end_date` from step 8
12. **Cap to top 5 + `OTHER`**
    - Keep the first 5 rows from the sorted full list
    - If more than 5 categories have activity:
      - `other_amount` = sum of `amount` for ranks 6+
      - `other_percent` = `other_amount / total_spend` when `total_spend > 0`, else `0`
      - Append `{ category: "OTHER", amount: other_amount, percent: other_percent }` as the **last** entry in `categories[]` (do not re-sort)
    - If 5 or fewer categories have activity: omit `OTHER`
    - Emit capped `categories[]` (max 6 entries: 5 named + `OTHER`)
13. **`as_of`**
    - `window_end`
14. **Format output**
    - Apply [output formatting](../../SKILL.md#output-formatting): dollar fields (`amount`, `total_spend`) to 2 dp; fraction percent fields (`percent`) to 3 dp

### Data output

**Formatting:** Dollar fields — 2 dp; fraction percent fields — 3 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `month` | string | Selected `YYYY-MM` |
| `start_date` | date | First calendar day of selected month |
| `end_date` | date | Last calendar day of selected month; partial latest month uses latest transaction date |
| `total_spend` | number | Sum of all category amounts for the month (uncapped) — not limited to top 5 + `OTHER` |
| `categories` | array | `{ category, amount, percent }` — top 5 by `amount` descending, plus optional `OTHER` last; max 6 entries |
| `categories[].percent` | number | Share of `total_spend` as a fraction (0–1) — 3 dp |
| `months` | `string[]` | Month picker options: up to 12 `YYYY-MM` values, ascending; includes zero-activity months |
| `as_of` | date | Latest transaction date on scoped, eligible rows |

**Date scope:** `months[]` lists selectable months as `YYYY-MM` strings only. Top-level `month` / `start_date` / `end_date` scope the category view for the selected month.

### Notes

- **`OTHER`** is a product rollup label, not a Plaid `personal_finance_category_primary` value.
- The top 5 categories are computed **per selected month** — which categories appear in the top 5 can change when the user picks a different month from `months[]`.
- `total_spend` reflects the full month total; `categories[]` is a capped view for charting (top 5 + remainder).
