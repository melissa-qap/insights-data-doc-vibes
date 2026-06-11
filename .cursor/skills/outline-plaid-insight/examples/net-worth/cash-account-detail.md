# Cash account detail

### Description

Single depository account header, available balance and subtype, and recent posted transactions. Scoped to one `account_id` where `type = depository`.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `account_id` | Selected account |
| `name`, `mask`, `official_name` | Header display |
| `item_id` | Join to institution |
| `subtype` | Account type label (e.g. checking) |
| `balances_available` | Available balance — may be null |
| `synced_at` | "Updated X ago" |

**Input:** `user_id = ?`, `account_id = ?`. Latest snapshot. `type = depository`.

#### `plaid_items` *(extension)*

| Column | Description |
|---|---|
| `institution_name` | Header institution label |

#### `plaid_transactions`

| Column | Description |
|---|---|
| `transaction_id`, `account_id`, `amount`, `date`, `name`, `merchant_name`, `pending` | Recent activity rows |

**Input:** Filter `account_id = ?`. Exclude `pending = true` and `removed = true`.

**Parameters:**

| Parameter | Default |
|---|---|
| `transaction_limit` | 20 |

### Calculation / analysis

1. **Load account** — latest row for `account_id`; reject if not `depository`
2. **Header** — `{ title: name + mask, institution_name, synced_at }`
3. **Info block** — `balances_available`, formatted `subtype`
4. **Account owner** — **Not available in current Plaid schema.** Omit from UI.
5. **Transactions** — sort by `date` desc; cap at `transaction_limit`; `display_name = COALESCE(merchant_name, name)`

### Data output

| Field | Type | Description |
|---|---|---|
| `account_id` | string | Selected account |
| `title` | string | `{name} • {mask}` or `name` |
| `institution_name` | string | From `plaid_items` |
| `synced_at` | timestamp | Last sync |
| `balances_available` | number \| null | May be null |
| `subtype` | string | Formatted account type |
| `transactions[]` | array | `{ transaction_id, display_name, date, amount }` |

### UI output

**Pattern:** [Account detail — flat table](../../ui-output-options.md#account-detail--flat-table) — header block + transaction rows.
