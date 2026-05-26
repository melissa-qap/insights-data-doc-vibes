# Investment accounts by institution

### Description

Lists all linked investment accounts grouped by financial institution, with each account's current balance and institution subtotals, as of the latest account sync.

### Required input data

#### `plaid_items` *(extension)*

Source: Plaid Item metadata at link time (not a standard Plaid datatable row today). One row per Item per user.

| Column | Description |
|---|---|
| `user_id` | User scope |
| `item_id` | Join key — matches `plaid_accounts.item_id` |
| `institution_name` | Institution display name (e.g. "Fidelity", "Vanguard") |

**Input:** `user_id = ?`. One row per `item_id` for the user.

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `item_id` | Links account to institution via `plaid_items` |
| `account_id` | Unique account identifier |
| `name` | Account display name |
| `official_name` | Official name from institution — optional display fallback |
| `mask` | Last 2–4 digits — optional display suffix |
| `type` | Account type — must be `investment` |
| `subtype` | Account subtype (e.g. `401k`, `ira`, `brokerage`) |
| `balances_current` | Current account balance — preferred balance source |
| `synced_at` | Snapshot timestamp for latest account state |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX` for user). `type = investment`. One row per `account_id`.

#### `plaid_investment_holdings`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Investment account holding positions |
| `institution_value` | Market value per position — summed when account balance is null |
| `synced_at` | Must match accounts snapshot for fallback sum |

**Input:** `user_id = ?`. Latest snapshot (same `synced_at` as `plaid_accounts`). Used only for accounts where `balances_current` IS NULL. Exclude rows where `institution_value` IS NULL when summing.

### Calculation / analysis

1. Load latest snapshot: `as_of` = `MAX(synced_at)` from `plaid_accounts` for the user.
2. Load `plaid_items` for the user. Join `plaid_accounts` on `item_id`.
3. Filter to investment accounts at `synced_at = as_of` (`type = investment`).
4. **Per-account balance** — for each account:
   - If `balances_current` IS NOT NULL: `balance = balances_current`, `balance_source = account`.
   - Else: `balance = SUM(institution_value)` from `plaid_investment_holdings` for that `account_id` at `as_of`; `balance_source = holdings`. Exclude null `institution_value` from the sum.
   - If balance is still null or the holdings sum is zero with no rows, **exclude** the account from output.
5. **Build account rows** — `{ account_id, name, mask, subtype, balance, balance_source, item_id, institution_name }` where `name` prefers `plaid_accounts.name`; append ` ••{mask}` in the UI layer when `mask` is present. `institution_name` from `plaid_items`; if missing, use `"Unknown institution"`.
6. **Group by institution** — group accounts by `item_id`. For each group:
   - `institution_name` from `plaid_items` (same for all accounts in the group)
   - `accounts[]` = account rows in the group
   - `total_balance` = sum of `accounts[].balance` (derived from accounts, not computed separately)
7. **Sort** — institutions by `total_balance` descending; accounts within each institution by `balance` descending.

**Portfolio total:** Not included in this insight. Combined investment value is provided by [investment performance chart](investment-performance-chart.md) (`end_value` at the active timeframe end) and displayed alongside this list in the UI.

### Data output

| Field | Type | Description |
|---|---|---|
| `as_of` | timestamp | `synced_at` of the snapshot used |
| `institutions` | array | Institution groups, sorted by `total_balance` descending |
| `institutions[].item_id` | string | Plaid Item identifier |
| `institutions[].institution_name` | string | Display name from `plaid_items` |
| `institutions[].total_balance` | number | Sum of child account balances (derived from `accounts`) |
| `institutions[].accounts` | array | `{ account_id, name, mask, subtype, balance, balance_source }` — sorted by `balance` descending |

### UI output

**Pattern:** [Nested list](../../ui-output-options.md#investment-accounts-by-institution--nested-list) — institutions at the top level, individual accounts nested below. Pair with [investment performance chart](investment-performance-chart.md) for the portfolio total.
