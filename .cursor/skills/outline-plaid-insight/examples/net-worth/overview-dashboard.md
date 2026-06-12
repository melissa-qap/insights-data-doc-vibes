# Overview dashboard

### Description

Composite overview screen: net worth headline and chart, assets/liabilities summary bar, and account list grouped by Cash, Investment, Credit cards, and Loans. Combines [net worth balance chart](net-worth-balance-chart.md), [net worth snapshot](net-worth-snapshot.md), [assets / liabilities bar](assets-liabilities-bar.md), and institution metadata from `plaid_items`. Net worth insights share [net worth core](net-worth-core.md) — fetch historical `plaid_accounts` once per request.

### Required input data

#### `plaid_accounts`

Same columns as [net worth snapshot](net-worth-snapshot.md) plus `mask`, `item_id`, `official_name`, `synced_at` for account row display and "Updated X ago".

#### `plaid_items` *(extension)*

| Column | Description |
|---|---|
| `item_id` | Join key — matches `plaid_accounts.item_id` |
| `institution_name` | Institution display label per account row |

**Input:** `user_id = ?`. All historical account rows (same scope as chart). Join on `item_id` when enriching account rows.

#### `plaid_investment_holdings`

Used only for investment account balance fallback when `balances_current` IS NULL — same rule as [investment accounts by institution](../investment-account/investment-accounts-by-institution.md).

**Parameters:**

| Parameter | Values |
|---|---|
| `timeframe` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` — passed to net worth balance chart |

**Timeframe aliases:** `1M` → `trailing_1m`; `3M` → `trailing_3m`; `YTD` → `ytd`; `1Y` → `trailing_1y`; `All` → `all_time`.

### Calculation / analysis

1. **Load accounts** — fetch all `plaid_accounts` rows for user (historical syncs); shared by chart and snapshot via [net worth core](net-worth-core.md)
2. **Net worth chart** — run [net worth balance chart](net-worth-balance-chart.md) with active `timeframe` (resolver loop + `compute_net_worth` with `include_groups = false`)
3. **Account list** — run [net worth snapshot](net-worth-snapshot.md) at `window_end` (single resolver call + `compute_net_worth` with `include_groups = true`); `snapshot.net_worth` must equal `chart.points[last].net_worth`
4. **Enrich account rows** — for each account in `asset_groups` / `liability_groups`:
   - Join `plaid_items` on `item_id` → `institution_name` (fallback `"Unknown institution"`)
   - Add `mask` from `plaid_accounts`; `synced_at` from snapshot account row (per-account, from core resolver)
   - Display label: `{name} • {mask}` when `mask` present; else `name` (prefer over `official_name`)
   - Investment accounts: if `balance` null, sum `plaid_investment_holdings.institution_value` at same `synced_at`
5. **Assets / liabilities bar** — run [assets / liabilities bar](assets-liabilities-bar.md) at `window_end` (same cutoff as snapshot); `bar.total_assets` and `bar.total_liabilities` must match snapshot totals; `bar.net_worth` must equal `chart.points[last].net_worth`
6. **Headline** — `points[last].net_worth` from chart; YTD label uses chart with `timeframe = ytd` → `period_return_amount`, `period_return_pct`

### Data output

| Field | Type | Description |
|---|---|---|
| `chart` | object | Full [net worth balance chart](net-worth-balance-chart.md) payload |
| `snapshot` | object | Full [net worth snapshot](net-worth-snapshot.md) payload |
| `bar` | object | Full [assets / liabilities bar](assets-liabilities-bar.md) payload (`segments`, `total_assets`, `total_liabilities`, `net_worth`, `bar_total`, `as_of`) |
| `accounts[]` | array | Enriched leaves: `{ account_id, name, mask, institution_name, balance, group, synced_at }` |
