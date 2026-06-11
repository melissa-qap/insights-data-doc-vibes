# Recurring investments

### Description

Lists all **active** recurring investment activity across linked investment accounts, detected from transaction patterns over the last 6 months. Each row shows the security, account, typical amount, detected frequency (weekly through annual, including semi-monthly), and last occurrence.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account identifier |
| `name` | Account display name |
| `type` | Account type — must be `investment` |

**Input:** `user_id = ?`. `type = investment`. Used to filter transactions and display account names.

#### `plaid_investment_transactions`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account |
| `security_id` | Security (nullable for some cash/transfer rows) |
| `date` | Transaction date — used for interval detection and active filter |
| `name` | Transaction description — fallback label when no security |
| `amount` | Dollar amount (positive = cash out / purchase) |
| `type` | High-level type (`buy`, `sell`, `cash`, `transfer`, `fee`, `dividend`) |
| `subtype` | Specific subtype — required for filtering `dividend`, `cash`, and `transfer` |

**Input:** `user_id = ?`. `account_id` in investment account list. `date` within last 6 months. Include rows matching **eligible types** below. Exclude `sell` and `fee`.

**Eligible types:**

| `type` | `subtype` rule |
|---|---|
| `buy` | Any subtype (typically `buy`) |
| `dividend` | `reinvested dividend` only |
| `cash` | Contribution subtypes only: `contribution`, `deposit`, `rollover` (case-insensitive match) |
| `transfer` | Contribution subtypes only: `contribution`, `deposit`, `rollover` (case-insensitive match) |

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |

**Input:** `user_id = ?`. Left join on `security_id`. Use transaction `name` when `security_id` is null.

### Calculation / analysis

1. **Detection window**
   - Last 6 months
   - `window_start` = today − 6 months
   - `window_end` = today
2. **Load account scope**
   - Investment `account_id` values from `plaid_accounts` where `type = investment`
3. **Load transactions**
   - `plaid_investment_transactions` for those accounts in the window
   - Apply **eligible types** filter
   - Join `plaid_investment_securities` on `security_id`
4. **Group candidates**
   - By `account_id` + `security_id` (or transaction `name` if `security_id` is null) + `type`/`subtype` (e.g. `buy`/`buy`, `dividend`/`reinvested dividend`, `transfer`/`contribution`)
5. **Minimum group size**
   - Keep groups with **≥ 2** transactions
6. **Detect frequency**
   - Compute from the **median** gap in days between consecutive `date` values (sorted ascending)
   - Assign the first matching bucket:
     - **Weekly:** 5–9 days
     - **Biweekly:** 10–15 days
     - **Semi-monthly:** 16–22 days
     - **Monthly:** 23–40 days
     - **Quarterly:** 75–105 days
     - **Annual:** 340–380 days
     - Groups outside these ranges are **excluded** (no stable cadence)
7. **Active filter**
   - Keep only if `last_date` is within **1.5×** the bucket's typical interval:
     - Weekly: ≤ 14 days ago
     - Biweekly: ≤ 23 days ago
     - Semi-monthly: ≤ 33 days ago
     - Monthly: ≤ 60 days ago
     - Quarterly: ≤ 158 days ago
     - Annual: ≤ 570 days ago
8. **Typical amount**
   - Median of `amount` across occurrences in the group (use absolute value)
9. **Build flat rows**
   - `{ account_id, account_name, security_name, ticker_symbol, type, subtype, frequency, typical_amount, last_date, occurrence_count }`
10. **Sort rows**
    - By frequency order (weekly → biweekly → semi-monthly → monthly → quarterly → annual), then `typical_amount` descending

### Data output

| Field | Type | Description |
|---|---|---|
| `recurrences` | array | Flat list: `{ account_id, account_name, security_name, ticker_symbol, type, subtype, frequency, typical_amount, last_date, occurrence_count }` |
| `window_start` | date | Start of 6-month detection window |
| `window_end` | date | End of detection window |

### UI output

**Pattern:** [Flat table](../../ui-output-options.md#recurring-investments--flat-table) — columns `Account`, `Security`, `Frequency`, `Amount`, `Last date`.

### Notes

- Plaid has **no recurring flag** — recurrence is inferred from transaction history; patterns may be missed or mislabeled.
- **Semi-monthly** vs **biweekly** are distinguished by median gap (10–15 vs 16–22 days).
- **Cash dividends** (subtype `dividend`, `qualified dividend`, etc.) are excluded — only **reinvested** dividends count.
- **`cash` / `transfer`** without a contribution subtype are excluded (e.g. withdrawals, journals). Extend the subtype allowlist if institutions use other contribution labels.
