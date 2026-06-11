# Investment account detail

### Description

Single investment account: performance chart, info block, simplified holdings allocation (Stocks / Crypto / Cash), and recent investment activities.

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

#### `plaid_investment_transactions`

| Column | Description |
|---|---|
| `investment_transaction_id`, `account_id`, `date`, `name`, `amount`, `type`, `subtype`, `security_id` | Activity rows |

**Parameters:**

| Parameter | Values / default |
|---|---|
| `timeframe` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` |
| `activity_limit` | 20 |

### Calculation / analysis

1. **Performance chart** — adapt [investment performance chart](investment-performance-chart.md): filter all steps to single `account_id`; daily `total_value = balances_current` point-in-time
2. **Header** — same shape as [cash account detail](../net-worth/cash-account-detail.md)
3. **Available balance** — `balances_available`; if null, sum holdings where security `type = cash`
4. **Account owner** — **Not available in current Plaid schema.** Omit from UI.
5. **Holdings buckets** — at latest holdings `synced_at`, classify each holding's `institution_value` V:
   - **Stocks:** `type` in (`equity`, `etf`, `mutual fund`, `fixed income`, `derivative`, `other`) — full V unless enrichment splits an ETF (use [asset allocation](asset-allocation.md) equity/bonds weights rolled into Stocks)
   - **Crypto:** `holdings.unofficial_currency_code IS NOT NULL`, or enrichment `asset_class = crypto`
   - **Cash:** security `type = cash`
   - Unclassified fund value (no enrichment): add to **Stocks**
6. **Bucket totals** — `stocks_value`, `crypto_value`, `cash_value`; `account_total` = sum of buckets; each `percent = value / account_total`
7. **Activities** — filter `account_id`; sort `date` desc; cap at `activity_limit`

### Data output

| Field | Type | Description |
|---|---|---|
| `chart` | object | Per-account performance chart payload (`points`, `period_return_*`, window fields) |
| `title`, `institution_name`, `synced_at`, `subtype` | — | Header / info |
| `balances_available` | number \| null | Or cash-holdings fallback |
| `holdings_buckets[]` | array | `{ bucket: stocks\|crypto\|cash, value, percent }` |
| `activities[]` | array | `{ investment_transaction_id, name, date, amount, type, subtype }` |

### UI output

**Pattern:** [Investment account detail — composite](../../ui-output-options.md#investment-account-detail--composite) — line chart + segmented allocation bar + activity flat table.
