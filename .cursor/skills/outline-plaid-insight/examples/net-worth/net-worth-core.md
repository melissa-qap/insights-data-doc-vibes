# Net worth core (shared partial)

### Description

Shared calculation pipeline for net worth insights. Not a user-facing insight — consumed by [net worth snapshot](net-worth-snapshot.md), [net worth balance chart](net-worth-balance-chart.md), and [assets / liabilities bar](assets-liabilities-bar.md).

Two layers:

1. **`resolve_accounts_as_of`** — point-in-time account rows from `plaid_accounts`
2. **`compute_net_worth`** — classify, roll up, and optionally build category groups

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
| [Net worth snapshot](net-worth-snapshot.md) | Latest sync only, or derive latest from history | `MAX(synced_at)` for user |
| [Net worth balance chart](net-worth-balance-chart.md) | All rows where `synced_at <= window_end` (optionally bounded by `window_start` for performance) | `end_of_day(D)` for each calendar day `D` in window |

Both read the same `plaid_accounts` table ([plaid-api-schema.md](../../plaid-api-schema.md)). The difference is how many rows are loaded, not which columns.

**Dashboard optimization:** when snapshot and chart run together ([overview dashboard](overview-dashboard.md)), fetch historical rows once; snapshot = resolver at `window_end`, chart = resolver loop over days.

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

**Options:**

| Option | Default | Description |
|---|---|---|
| `include_groups` | `false` | When `true`, build `asset_groups` and `liability_groups` |

**Steps:**

1. **Classify by `type`**
   - **Assets:** `depository`, `investment`
   - **Liabilities:** `credit`, `loan`
   - **Other:** `other` — default to asset unless `subtype` indicates liability
2. **Assign group within `role`**
   - **Cash** (`group = cash`): `type = depository`, or (`type = other` and `role = asset`)
   - **Investment** (`group = investment`): `type = investment`
   - **Credit cards** (`group = credit_cards`): `type = credit`
   - **Loans** (`group = loans`): `type = loan`, or (`type = other` and `role = liability`)
3. **Build per-account list**
   - Enrich each resolved account with `role` and `group`; pass through `synced_at` from Layer 2
   - Output shape: `{ account_id, name, type, subtype, balance, synced_at, role, group }`
   - `synced_at` = timestamp of the row used for that account's balance (may differ across accounts)
   - Per schema: use `balance` (`balances_current`); do not use `balances_available`
4. **Derive category balances from the account list**
   - Do not compute independently
   - `cash_balance` = sum of `balance` where `group = cash` (0 if none)
   - `investment_balance` = sum of `balance` where `group = investment` (0 if none)
   - `credit_cards_balance` = sum of `balance` where `group = credit_cards` (0 if none)
   - `loans_balance` = sum of `balance` where `group = loans` (0 if none)
5. **Derive portfolio totals from category balances**
   - Do not compute independently
   - `total_assets` = `cash_balance` + `investment_balance`
   - `total_liabilities` = `credit_cards_balance` + `loans_balance`
   - `net_worth` = `total_assets` − `total_liabilities`
6. **Build category groups** *(when `include_groups = true`)*
   - `total_balance` on each group must equal its top-level category balance
   - **Asset groups** — `cash`, then `investment`: each `{ group, label, total_balance, accounts[] }` where `total_balance` is `cash_balance` or `investment_balance`; omit groups with no accounts
   - **Liability groups** — `credit_cards`, then `loans`: `total_balance` is `credit_cards_balance` or `loans_balance`; omit empty groups
   - Sort accounts within each group by `balance` descending

### Output contract

| Field | Type | When present |
|---|---|---|
| `accounts[]` | array | Always — `{ account_id, name, type, subtype, balance, synced_at, role, group }` |
| `cash_balance` | number | Always |
| `investment_balance` | number | Always |
| `credit_cards_balance` | number | Always |
| `loans_balance` | number | Always |
| `total_assets` | number | Always |
| `total_liabilities` | number | Always |
| `net_worth` | number | Always |
| `asset_groups` | array | When `include_groups = true` |
| `liability_groups` | array | When `include_groups = true` |
| `as_of` | timestamp | `MAX(synced_at)` across resolved `accounts[]` |

### Caller mapping

| Caller | Resolver cutoff | `include_groups` | Output used |
|---|---|---|---|
| Snapshot | `MAX(synced_at)` | `true` | Full payload + groups |
| Chart (per day) | `end_of_day(D)` | `false` | `net_worth` only → `{ date, net_worth }` |
| Assets / liabilities bar | `MAX(synced_at)` or `window_end` | `false` | `total_assets`, `total_liabilities`, `net_worth` → `segments[]` |

**Invariant (paired UI):** snapshot `net_worth` at latest sync must equal chart `points[last].net_worth` when both use the same `window_end`.

### Implementation notes

**Efficient chart series (avoid N full table scans):**

1. Load all `plaid_accounts` rows for user in `[window_start, window_end]`, ordered by `account_id`, `synced_at`
2. Build a sync-date change list per account (only dates where balance actually changes)
3. Forward-fill daily balances from last known sync ≤ D
4. Run `compute_net_worth` on the filled daily account set; skip group building (`include_groups = false`)

**Caching within a request:** `role` / `group` mapping is stable unless `type`/`subtype` changes at an as-of row — safe to compute once per account per as-of set.

**Validation:** assert `snapshot.net_worth = chart.points[last].net_worth` when both run on the overview dashboard.
