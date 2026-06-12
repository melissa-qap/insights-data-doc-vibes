# Investment account detail

### Description

Single investment account: performance chart, info block, simplified holdings allocation by Plaid security type (9 buckets), and recent investment activities.

### Required input data

#### `plaid_accounts`

| Column | Description |
|---|---|
| `user_id`, `account_id`, `name`, `mask`, `item_id`, `subtype` | Header and info |
| `balances_current`, `balances_available`, `synced_at` | Value and freshness |
| `type` | Must be `investment` |

#### `plaid_items` *(extension)*

| Column | Description |
|---|---|
| `institution_name` | Header institution |

#### `plaid_investment_holdings` + `plaid_investment_securities`

Same join as [asset allocation](asset-allocation.md), filtered to one `account_id`.

**Not available — API not linked.** `/investments/transactions/get` is not imported; omit `activities[]` from output (return empty array) until the endpoint is linked.

**Parameters:**

| Parameter | Values / default |
|---|---|
| `timeframe` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` |
| `activity_limit` | 20 |

### Calculation / analysis

1. **Performance chart** — adapt [investment performance chart](investment-performance-chart.md): filter all steps to single `account_id`; daily `total_value = balances_current` point-in-time
2. **Header** — same shape as [cash account detail](../net-worth/cash-account-detail.md)
3. **Available balance** — `balances_available`; if null, sum holdings where normalized security type = `cash` per [enum_investment_security_type](../../plaid-api-schema.md#enum_investment_security_type)
4. **Account owner** — **Not available in current Plaid schema.** Omit from output.
5. **Holdings buckets** — at latest holdings `synced_at`, classify each holding's `institution_value` V using the same normalization as [asset allocation](asset-allocation.md):
   - One of 9 Plaid security types — see [enum_investment_security_type](../../plaid-api-schema.md#enum_investment_security_type)
   - Omit buckets with zero value from output
6. **Bucket totals** — one entry per non-zero bucket: `{ bucket, value, percent }`; `account_total` = sum of bucket values; each `percent = value / account_total`
7. **Activities** — **blocked** (API not linked). When `/investments/transactions/get` is linked: filter `account_id`; sort `date` desc; cap at `activity_limit`. Until then, return `activities = []`.

### Data output

| Field | Type | Description |
|---|---|---|
| `chart` | object | Per-account performance chart payload (`points`, `period_return_*`, window fields) |
| `title`, `institution_name`, `synced_at`, `subtype` | — | Header / info |
| `balances_available` | number \| null | Or cash-holdings fallback |
| `holdings_buckets[]` | array | `{ bucket, value, percent }` — one of 9 Plaid security types; non-zero buckets only |
| `activities[]` | array | `{ investment_transaction_id, name, date, amount, type, subtype }` — empty when API not linked |
