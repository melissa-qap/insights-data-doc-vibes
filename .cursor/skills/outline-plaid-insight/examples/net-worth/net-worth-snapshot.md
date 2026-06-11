# Net worth snapshot

### Description

Shows per-account balances grouped under Cash, Investment, Credit cards, and Loans as of the latest account sync. Computes `net_worth` for validation and pairing with [net worth balance chart](net-worth-balance-chart.md). Category balances are shown in the nested list; Assets, Liabilities, and net worth are not — the chart supplies the headline total (`points[last].net_worth`).

Uses [net worth core](net-worth-core.md) with latest-only data access and full group output.

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
| `synced_at` | Per-account sync timestamp — resolved independently per `account_id` in [net worth core](net-worth-core.md) Layer 2 |

**Input:** `user_id = ?`. Latest snapshot (`synced_at = MAX` for user). One row per `account_id`. Exclude null `balances_current`.

Equivalent to [net worth core](net-worth-core.md) Layer 1 with cutoff = `MAX(synced_at)`.

### Calculation / analysis

1. **`cutoff`** — `MAX(synced_at)` for user
2. **Resolve accounts** — [net worth core](net-worth-core.md) Layer 2: `resolve_accounts_as_of(user_id, cutoff)`
3. **Compute net worth** — [net worth core](net-worth-core.md) Layer 3: `compute_net_worth(accounts, include_groups = true)`
4. **Map output** — use core output fields directly (`net_worth`, category balances, `asset_groups`, `liability_groups`, `as_of`); each account in groups includes its own `synced_at`

### Data output

| Field | Type | Description |
|---|---|---|
| `net_worth` | number | Total assets minus total liabilities — not shown in nested list UI; use chart `points[last].net_worth` or this value for the headline when paired |
| `total_assets` | number | `cash_balance` + `investment_balance` — for calculation/validation; not shown in nested list UI |
| `total_liabilities` | number | `credit_cards_balance` + `loans_balance` — for calculation/validation; not shown in nested list UI |
| `cash_balance` | number | Sum of cash-group account balances (derived from accounts) |
| `investment_balance` | number | Sum of investment-group account balances (derived from accounts) |
| `credit_cards_balance` | number | Sum of credit-card-group account balances (derived from accounts) |
| `loans_balance` | number | Sum of loan-group account balances (derived from accounts) |
| `as_of` | timestamp | Portfolio freshness — `MAX(synced_at)` across resolved accounts; may be newer than some per-account sync times |
| `asset_groups` | array | `{ group, label, total_balance, accounts[] }` — `cash`, `investment`; `total_balance` matches `cash_balance` / `investment_balance`; `accounts[]` = `{ account_id, name, type, subtype, balance, synced_at, role, group }` |
| `liability_groups` | array | `{ group, label, total_balance, accounts[] }` — `credit_cards`, `loans`; `total_balance` matches `credit_cards_balance` / `loans_balance`; same account shape |

### UI output

**Pattern:** [Nested list](../../ui-output-options.md#net-worth--nested-list) — Assets / Liabilities (headers only) → Cash / Investment or Credit cards / Loans (balance) → individual accounts (balance). Pair with [net worth balance chart](net-worth-balance-chart.md) for the net worth headline (`points[last].net_worth`); do not render `net_worth` on this list.
