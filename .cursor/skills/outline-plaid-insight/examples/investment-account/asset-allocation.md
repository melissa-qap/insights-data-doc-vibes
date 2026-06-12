# Asset allocation across investment accounts

### Description

Shows portfolio-wide asset allocation by Plaid security type (9 buckets) across all linked investment accounts as of the latest holdings snapshot. Each holding is normalized to a security type and assigned 100% of its market value to one bucket. Each bucket includes a holdings drill-down.

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
| `institution_value` | Current market value — basis for allocation |
| `synced_at` | Snapshot timestamp — must match securities join |

**Input:** `user_id = ?`. Latest snapshot (same `synced_at` as `plaid_accounts`). `account_id` in investment account list. Exclude rows where `institution_value` IS NULL.

#### `plaid_investment_securities`

| Column | Description |
|---|---|
| `user_id` | User scope |
| `security_id` | Join key to holdings |
| `name` | Security display name |
| `ticker_symbol` | Ticker for display |
| `type` | Security type — normalized to output bucket via [enum_investment_security_type](../../plaid-api-schema.md#enum_investment_security_type) |
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
4. **Classify each holding**
   - `institution_value` = V
   - `asset_class = normalize(securities.type, securities.subtype)` using [enum_investment_security_type](../../plaid-api-schema.md#enum_investment_security_type)
   - One slice per holding: `{ security_id, asset_class, value: V }`
5. **Roll up by `asset_class`**
   - `value` = sum of holding values in that class
   - `total_investment_value` = sum of all class values
6. **Holdings drill-down per class**
   - Group by `security_id` within each class (one row per security; full `institution_value`)
   - Use security `name`; fall back to `ticker_symbol` if null
   - Sort holdings within class by `value` descending
7. **Percents**
   - For each class, `percent` = `value / total_investment_value` (0 if denominator 0)
   - Sort `asset_classes` by `value` descending; omit classes with zero value
8. **`as_of`**
   - Snapshot `synced_at`

**Heuristic limitations:** Allocation reflects Plaid instrument type, not underlying fund composition (e.g. a bond ETF with `type = etf` counts as `etf`).

### Data output

| Field | Type | Description |
|---|---|---|
| `asset_classes` | array | One entry per class with non-zero value: `{ asset_class, value, percent, holdings }` — sorted by `value` descending |
| `asset_classes[].asset_class` | string | One of 9 Plaid security types — see [enum_investment_security_type](../../plaid-api-schema.md#enum_investment_security_type) |
| `asset_classes[].value` | number | Class total — sum of `holdings[].value` |
| `asset_classes[].percent` | number | Share of `total_investment_value` as a fraction (0–1) |
| `asset_classes[].holdings` | array | `{ security_id, name, ticker_symbol, value }` |
| `total_investment_value` | number | Sum of all class values |
| `as_of` | timestamp | `synced_at` of the snapshot used |
