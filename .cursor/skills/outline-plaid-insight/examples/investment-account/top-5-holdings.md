# Top 5 holdings

### Description

Shows the user's five largest investment positions by market value across all linked investment accounts, with a per-account breakdown under each holding.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier — filter to investment accounts |
| `name` | Account display name — used in per-account breakdown |
| `type` | Account type — must be `investment` |
| `synced_at` | Snapshot timestamp — align with holdings sync |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX` for user). `type = investment`. Collect `account_id` list for holdings filter.

#### `plaid_investment_holdings`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account holding the position |
| `security_id` | Links to security metadata |
| `quantity` | Shares/units held |
| `institution_value` | Current market value — used for ranking |
| `synced_at` | Snapshot timestamp — must match securities join |

**Input:** `user_id = ?`. Latest snapshot (same `synced_at` as `plaid_accounts`). `account_id` in investment account list. Exclude rows where `institution_value` IS NULL.

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key to holdings |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |
| `type` | Security type (`equity`, `etf`, `mutual fund`, etc.) |
| `synced_at` | Must match holdings `synced_at` on join |

**Input:** `user_id = ?`. Latest snapshot. Join to holdings on `security_id` + `synced_at`.

### Calculation / analysis

1. Load latest snapshot timestamp from `plaid_accounts` for the user.
2. Get all `account_id` values where `type = investment`.
3. Load `plaid_investment_holdings` for those accounts at that `synced_at`. Join `plaid_investment_securities` on `security_id` + `synced_at`.
4. **Build per-account rows** — for each holding row, `{ account_id, account_name, value }` where `value = institution_value` and `account_name` comes from `plaid_accounts.name`.
5. **Aggregate by security** — group by `security_id`; collect account rows into `accounts[]`. Derive `total_value` = sum of `accounts[].value` (do not compute independently).
6. **Rank** — sort securities by `total_value` descending; take top 5.
7. **Sort accounts within each holding** — order `accounts` by `value` descending (largest position first).
8. **Build holdings list** — `{ rank, security_id, name, ticker_symbol, total_value, security_type, accounts }` for ranks 1–5. Use security `name` from `plaid_investment_securities`; fall back to `ticker_symbol` if null.

### Data output

| Field | Type | Description |
|---|---|---|
| `holdings` | array | Top 5 positions: `{ rank, security_id, name, ticker_symbol, total_value, security_type, accounts }` — ordered by `rank` (1 = largest) |
| `holdings[].accounts` | array | Per-account breakdown: `{ account_id, account_name, value }` — sorted by `value` descending; sums to `total_value` |
| `as_of` | timestamp | `synced_at` of the snapshot used |

### UI output

**Pattern:** [Nested list](../../ui-output-options.md#top-5-holdings--nested-list) — holding at top level with `total_value`; accounts nested below with per-account `value`.
