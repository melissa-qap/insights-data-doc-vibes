# Net worth core (shared partial)

### Description

Shared calculation pipeline for net worth read APIs and composite insights. Not a user-facing insight — consumed by [net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md) and [overview dashboard](overview-dashboard.md).

Two layers:

1. **`resolve_accounts_as_of`** — point-in-time account rows from `plaid_accounts`
2. **`compute_net_worth`** — classify accounts and roll up portfolio totals

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
| `synced_at` | Snapshot timestamp |

**Query scope** is caller-specific — see [Layer 1 — Data access](#layer-1--data-access).

### Layer 1 — Data access

| Caller | Query scope | Cutoff |
|---|---|---|
| [GET /v1/account-balance](../../design-api/examples/net-worth/net-worth-apis.md#get-v1account-balance) | Latest sync only, or derive latest from history | `MAX(synced_at)` for user (or `end_of_day(as_of)`) |
| [GET /v1/performance-history](../../design-api/examples/net-worth/net-worth-apis.md#get-v1performance-history) | All rows where `synced_at <= window_end` (optionally bounded by `window_start` for performance) | `end_of_day(D)` for each calendar day `D` in window |
| [GET /v1/assets-liabilities](../../design-api/examples/net-worth/net-worth-apis.md#get-v1assets-liabilities) | Latest sync only | Same as account-balance |

All read from the same `plaid_accounts` table ([plaid-api-schema.md](../../plaid-api-schema.md)). The difference is how many rows are loaded, not which columns.

**Dashboard optimization:** when overview screen loads all three read APIs ([overview dashboard](overview-dashboard.md)), fetch historical rows once; account-balance and bar = resolver at latest cutoff, performance-history = resolver loop over days.

### Layer 2 — `resolve_accounts_as_of`

**Input:** `user_id`, `cutoff` (timestamp or end-of-day)

**Logic:**

1. For each `account_id`, take the row with greatest `synced_at` where `synced_at <= cutoff`
2. Exclude rows with null `balances_current`
3. Omit accounts with no row on or before `cutoff`

**Output:** `accounts[]` — one entry per included account:

| Field | Source |
|---|---|
| `account_id` | `account_id` |
| `name` | `name` |
| `type` | `type` |
| `subtype` | `subtype` |
| `balance` | `balances_current` |
| `synced_at` | `synced_at` of the row selected |

**Metadata rule:** take `type`, `subtype`, and `name` from the **same as-of row** as `balance` (not from a separate latest-metadata join), so reclassification tracks account changes over time.

**SQL pattern** (matches schema point-in-time query):

```sql
-- For each account_id, latest row at or before cutoff with non-null balance
SELECT DISTINCT ON (account_id)
  account_id, name, type, subtype,
  balances_current AS balance,
  synced_at
FROM plaid_accounts
WHERE user_id = ?
  AND synced_at <= ?
  AND balances_current IS NOT NULL
ORDER BY account_id, synced_at DESC
```

**Shortcut for latest snapshot:** `cutoff = MAX(synced_at)` for user is equivalent to the schema's current-state query (`synced_at = MAX` per user).

**Carry-forward (chart):** consecutive calendar days share the same resolved account set until the next sync on or before that day. Calling the resolver at `end_of_day(D)` implements this automatically.

### Layer 3 — `compute_net_worth`

**Input:** resolved `accounts[]` from Layer 2

**Steps:**

1. **Classify by `type`**
   - **Assets:** `depository`, `investment`
   - **Liabilities:** `credit`, `loan`
   - **Other:** `other` — default to asset unless `subtype` indicates liability
   - Store result as `a_l`: `asset` or `liability`
2. **Assign group within `a_l`**
   - **Cash** (`group = cash`): `type = depository`, or (`type = other` and `a_l = asset`)
   - **Investment** (`group = investment`): `type = investment`
   - **Credit cards** (`group = credit_cards`): `type = credit`
   - **Loans** (`group = loans`): `type = loan`, or (`type = other` and `a_l = liability`)
3. **Build per-account list**
   - Enrich each resolved account with `a_l` and `group`; pass through `synced_at` from Layer 2
   - Output shape: `{ account_id, name, type, subtype, balance, synced_at, a_l, group }`
   - `synced_at` = timestamp of the row used for that account's balance (may differ across accounts)
   - Per schema: use `balance` (`balances_current`); do not use `balances_available`
4. **Derive portfolio totals from the account list**
   - Do not compute independently
   - `total_assets` = sum of `balance` where `a_l = asset` (0 if none)
   - `total_liabilities` = sum of `balance` where `a_l = liability` (0 if none)
   - `net_worth` = `total_assets` − `total_liabilities`
5. **Format output**
   - Apply [output formatting](../../SKILL.md#output-formatting): round all monetary output fields to 2 dp (`balance`, portfolio totals)

### Output contract

**Formatting:** Dollar fields — 2 dp ([SKILL.md output formatting](../../SKILL.md#output-formatting)).

| Field | Type | When present |
|---|---|---|
| `accounts[]` | array | Always — `{ account_id, name, type, subtype, balance, synced_at, a_l, group }` |
| `total_assets` | number | Always |
| `total_liabilities` | number | Always |
| `net_worth` | number | Always |
| `as_of` | timestamp | `MAX(synced_at)` across resolved `accounts[]` |

### Caller mapping

| API route | Resolver cutoff | Output used |
|---|---|---|
| `GET /v1/account-balance` | `MAX(synced_at)` or `end_of_day(as_of)` | `accounts[]` with `a_l`, `group`; optional filter by `account_ids[]` |
| `GET /v1/performance-history` (net worth mode, per day) | `end_of_day(D)` | `net_worth` → `{ date, value }` |
| `GET /v1/performance-history` (accounts mode, per day) | `end_of_day(D)` | Filter to `account_ids`, then `net_worth` of subset → `{ date, value }` |
| `GET /v1/assets-liabilities` | `MAX(synced_at)` or `end_of_day(as_of)` | `total_assets`, `total_liabilities`, `net_worth` → `segments[]` |

Full route specs: [net worth APIs](../../design-api/examples/net-worth/net-worth-apis.md).

**Cross-API invariant:** when account-balance returns all accounts (no `account_ids` filter), `sum(accounts where a_l=asset) − sum(accounts where a_l=liability)` at latest sync must equal `performance-history.points[last].value` and `assets-liabilities.net_worth`.

### Implementation notes

**Efficient chart series (avoid N full table scans):**

1. Load all `plaid_accounts` rows for user in `[window_start, window_end]`, ordered by `account_id`, `synced_at`
2. Build a sync-date change list per account (only dates where balance actually changes)
3. Forward-fill daily balances from last known sync ≤ D
4. Run `compute_net_worth` on the filled daily account set

**Caching within a request:** `a_l` / `group` mapping is stable unless `type`/`subtype` changes at an as-of row — safe to compute once per account per as-of set.

**Validation:** on net worth page load, client rollup over account-balance `accounts[]` must match `performance-history.points[last].value` and `assets-liabilities` totals.
