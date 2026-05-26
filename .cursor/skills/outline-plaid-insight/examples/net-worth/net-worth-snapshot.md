# Net worth snapshot

### Description

Shows per-account balances grouped under Cash, Investment, Credit cards, and Loans as of the latest account sync. Computes `net_worth` for validation and pairing with [net worth balance chart](net-worth-balance-chart.md). Category balances are shown in the nested list; Assets, Liabilities, and net worth are not — the chart supplies the headline total (`end_net_worth`).

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Unique account identifier |
| `name` | Account display name |
| `type` | Account type (`depository`, `credit`, `investment`, `loan`, `other`) |
| `subtype` | Account subtype (e.g. `checking`, `credit card`) |
| `balances_current` | Current balance — used for net worth |
| `synced_at` | Snapshot timestamp for latest account state |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX` for user). One row per `account_id`. Exclude null `balances_current`.

### Calculation / analysis

1. Load the latest account snapshot per the schema's current-state query pattern.
2. For each account, classify by `type`:
   - **Assets:** `depository`, `investment`
   - **Liabilities:** `credit`, `loan`
   - **Other:** `other` — default to asset unless `subtype` indicates liability
3. Assign a **group** within `role`:
   - **Cash** (`group = cash`): `type = depository`, or (`type = other` and `role = asset`)
   - **Investment** (`group = investment`): `type = investment`
   - **Credit cards** (`group = credit_cards`): `type = credit`
   - **Loans** (`group = loans`): `type = loan`, or (`type = other` and `role = liability`)
4. For each account, use `balances_current` (per schema: use `current` for net worth; do not use `balances_available`).
5. Build the **per-account list** — one entry per account: `{ account_id, name, type, subtype, balance, role, group }` where `balance = balances_current`.
6. **Derive category balances from the account list** (do not compute independently):
   - `cash_balance` = sum of `balance` where `group = cash` (0 if none)
   - `investment_balance` = sum of `balance` where `group = investment` (0 if none)
   - `credit_cards_balance` = sum of `balance` where `group = credit_cards` (0 if none)
   - `loans_balance` = sum of `balance` where `group = loans` (0 if none)
7. **Build category groups** from the account list — `total_balance` on each group must equal its top-level category balance:
   - **Asset groups** — `cash`, then `investment`: each `{ group, label, total_balance, accounts[] }` where `total_balance` is `cash_balance` or `investment_balance`; omit groups with no accounts
   - **Liability groups** — `credit_cards`, then `loans`: `total_balance` is `credit_cards_balance` or `loans_balance`; omit empty groups
   - Sort accounts within each group by `balance` descending
8. **Derive portfolio totals from category balances** (do not compute independently):
   - `total_assets` = `cash_balance` + `investment_balance`
   - `total_liabilities` = `credit_cards_balance` + `loans_balance`
   - `net_worth` = `total_assets` − `total_liabilities`

### Data output

| Field | Type | Description |
|---|---|---|
| `net_worth` | number | Total assets minus total liabilities — not shown in nested list UI; use chart `end_net_worth` for the headline when paired |
| `total_assets` | number | `cash_balance` + `investment_balance` — for calculation/validation; not shown in nested list UI |
| `total_liabilities` | number | `credit_cards_balance` + `loans_balance` — for calculation/validation; not shown in nested list UI |
| `cash_balance` | number | Sum of cash-group account balances (derived from accounts) |
| `investment_balance` | number | Sum of investment-group account balances (derived from accounts) |
| `credit_cards_balance` | number | Sum of credit-card-group account balances (derived from accounts) |
| `loans_balance` | number | Sum of loan-group account balances (derived from accounts) |
| `as_of` | timestamp | `synced_at` of the snapshot used |
| `asset_groups` | array | `{ group, label, total_balance, accounts[] }` — `cash`, `investment`; `total_balance` matches `cash_balance` / `investment_balance`; `accounts[]` = `{ account_id, name, type, subtype, balance, role, group }` |
| `liability_groups` | array | `{ group, label, total_balance, accounts[] }` — `credit_cards`, `loans`; `total_balance` matches `credit_cards_balance` / `loans_balance`; same account shape |

### UI output

**Pattern:** [Nested list](../../ui-output-options.md#net-worth--nested-list) — Assets / Liabilities (headers only) → Cash / Investment or Credit cards / Loans (balance) → individual accounts (balance). Pair with [net worth balance chart](net-worth-balance-chart.md) for the net worth headline (`end_net_worth`); do not render `net_worth` on this list.
