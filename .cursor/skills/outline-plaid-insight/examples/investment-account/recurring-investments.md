# Recurring investments

### Description

Lists all **active** recurring investment activity across linked investment accounts, detected from transaction patterns over up to **24 months** of available data (required for annual and quarterly cadences). Each row shows the security, account, typical amount, detected frequency (weekly through annual, including semi-monthly), and last occurrence.

**Not available — API not linked.** This insight requires `plaid_investment_transactions`, sourced from `/investments/transactions/get`, which is not imported in the current schema. Return an empty `recurrences` array until the endpoint is linked.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account identifier |
| `name` | Account display name |
| `type` | Account type — must be `investment` |

**Input:** `user_id = ?`. `type = investment`. Used to filter transactions and display account names.

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |

**Input:** `user_id = ?`. Left join on `security_id`. Use transaction `name` when `security_id` is null.

### Calculation / analysis

1. **Windows**
   - `window_end` = today
   - **Display window** — last 6 months (metadata only; does not limit which transactions are loaded)
     - `window_start` = today − 6 months
2. **Load account scope**
   - Investment `account_id` values from `plaid_accounts` where `type = investment`
3. **Load transactions**
   - `plaid_investment_transactions` for those accounts
   - Apply **eligible types** filter
   - Join `plaid_investment_securities` on `security_id`
4. **Detection lookback** — at most 24 months, but not before earliest available data (needed so annual patterns have ≥ 2 occurrences ~12 months apart)
   - `lookback_cap_start` = today − 24 months (matches Plaid investment history max)
   - `earliest_transaction_date` = minimum `date` on loaded rows from step 3
   - `detection_lookback_start` = **later** of `lookback_cap_start` and `earliest_transaction_date` — i.e. use all available in-scope history up to 24 months; do not look back before the first transaction
5. **Filter by detection lookback**
   - Keep rows where `date` is on or between `detection_lookback_start` and `window_end` (inclusive)
6. **Group candidates**
   - By `account_id` + `security_id` (or transaction `name` if `security_id` is null) + `type`/`subtype` (e.g. `buy`/`buy`, `dividend`/`reinvested dividend`, `transfer`/`contribution`)
7. **Minimum group size**
   - Keep groups with **≥ 2** transactions
8. **Detect frequency**
   - Compute from the **median** gap in days between consecutive `date` values (sorted ascending)
   - Assign the first matching bucket:
     - **Weekly:** 5–9 days
     - **Biweekly:** 10–15 days
     - **Semi-monthly:** 16–22 days
     - **Monthly:** 23–40 days
     - **Quarterly:** 75–105 days
     - **Annual:** 340–380 days
     - Groups outside these ranges are **excluded** (no stable cadence)
9. **Active filter**
   - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
     - Weekly: ≤ 14 days ago
     - Biweekly: ≤ 23 days ago
     - Semi-monthly: ≤ 33 days ago
     - Monthly: ≤ 60 days ago
     - Quarterly: ≤ 158 days ago
     - Annual: ≤ 570 days ago
10. **Typical amount**
   - Median of `amount` across occurrences in the group (use absolute value)
11. **Build flat rows**
   - `{ account_id, account_name, security_name, ticker_symbol, type, subtype, frequency, typical_amount, last_date, occurrence_count }`
12. **Sort rows**
    - By frequency order (weekly → biweekly → semi-monthly → monthly → quarterly → annual), then `typical_amount` descending
13. **Format output**
    - Apply [output formatting](../../SKILL.md#output-formatting): round all monetary fields (`typical_amount`) to 2 dp

### Data output

**Formatting:** Dollar fields — 2 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `recurrences` | array | Flat list: `{ account_id, account_name, security_name, ticker_symbol, type, subtype, frequency, typical_amount, last_date, occurrence_count }` |
| `detection_lookback_start` | date | Start of transaction lookback for pattern detection — later of (today − 24 months) and earliest in-scope transaction `date` |
| `window_start` | date | Start of 6-month display window |
| `window_end` | date | End of detection and display window (today) |

### Notes

- **API not linked:** `/investments/transactions/get` is not imported — this insight cannot produce results until the endpoint is linked and `plaid_investment_transactions` is populated.
- **Detection lookback vs display window:** Transactions are loaded from `detection_lookback_start` through `window_end` — at most **24 months**, floored at the earliest available in-scope transaction. Annual contributions need ≥ 2 occurrences ~365 days apart, so annual patterns are still missed when actual history spans less than ~13 months. `window_start` / `window_end` (6 months) are display-window metadata only.
- Plaid has **no recurring flag** — recurrence is inferred from transaction history; patterns may be missed or mislabeled.
- **Semi-monthly** vs **biweekly** are distinguished by median gap (10–15 vs 16–22 days).
- **Cash dividends** (subtype `dividend`, `qualified dividend`, etc.) are excluded — only **reinvested** dividends count.
- **`cash` / `transfer`** without a contribution subtype are excluded (e.g. withdrawals, journals). Extend the subtype allowlist if institutions use other contribution labels.
