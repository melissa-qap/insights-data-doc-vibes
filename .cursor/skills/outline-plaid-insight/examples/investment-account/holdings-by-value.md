# Holdings by value

### Description

Shows all investment positions merged by security and sorted by total market value across linked investment accounts as of the latest holdings snapshot. When the same security appears in multiple accounts, market value is summed into one row.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Account identifier — filter to investment accounts |
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
| `institution_value` | Current market value — used for sorting |
| `synced_at` | Snapshot timestamp — must match securities join |

**Input:** `user_id = ?`. Latest snapshot (same `synced_at` as `plaid_accounts`). `account_id` in investment account list. Exclude rows where `institution_value` IS NULL.

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key to holdings |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |
| `synced_at` | Must match holdings `synced_at` on join |

**Input:** `user_id = ?`. Latest snapshot. Join to holdings on `security_id` + `synced_at`.

### Calculation / analysis

1. **Load snapshot timestamp**
   - Latest snapshot from `plaid_accounts` for the user
2. **Load investment accounts**
   - All `account_id` values where `type = investment`
3. **Load holdings**
   - `plaid_investment_holdings` for those accounts at that `synced_at`
   - Join `plaid_investment_securities` on `security_id` + `synced_at`
4. **Aggregate by security**
   - Group by `security_id`
   - `total_value` = sum of `institution_value` across all investment accounts for that security
   - Attach `name` and `ticker_symbol` from `plaid_investment_securities`; fall back to `ticker_symbol` if `name` is null
5. **Sort**
   - Sort all securities by `total_value` descending
6. **Build holdings list**
   - `{ security_id, ticker_symbol, name, total_value }` for each security
7. **Format output**
   - Apply [output formatting](../../SKILL.md#output-formatting): round `total_value` to 2 dp

### Data output

**Formatting:** Dollar fields — 2 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `holdings` | array | `{ security_id, ticker_symbol, name, total_value }` — sorted by `total_value` descending |
| `holdings[].security_id` | string | Plaid security identifier |
| `holdings[].ticker_symbol` | string | Ticker for display |
| `holdings[].name` | string | Security display name |
| `holdings[].total_value` | number | Market value summed across all investment accounts |
| `as_of` | timestamp | `synced_at` of the snapshot used |
