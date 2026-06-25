# Investment performance chart

### Description

Shows combined investment account value as a daily time series for a selected timeframe, with simple holding-period return ($ and %) for that window. Suitable for a line chart of portfolio value over time with a period performance label.

Client wrapper around `GET /v1/performance-history` — backend resolves all investment `account_id` values and passes them as `account_ids`. Maps canonical `value` → `total_value` in output.

### Required input data

Same as [GET /v1/performance-history](../../../design-api/examples/net-worth/net-worth-apis.md#get-v1performance-history) — `plaid_accounts` with historical syncs. Wrapper additionally reads latest snapshot to collect investment account IDs.

**Parameters:**

| Parameter | Type | Default | Notes / options |
|---|---|---|---|
| `timeframe` | enum | `trailing_1y` | `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` |

### Calculation / analysis

1. **Resolve investment account IDs**
   - At `MAX(synced_at)` for user, collect all `account_id` values where `type = investment` and `balances_current` IS NOT NULL
2. **Call performance history** — `GET /v1/performance-history` with that `account_ids` list and the same `timeframe` ([net worth APIs](../../../design-api/examples/net-worth/net-worth-apis.md#get-v1performance-history))
3. **Map output**
   - `points[].total_value` ← `points[].value`
   - `start_value` = `points[0].total_value`; `end_value` = `points[last].total_value`
   - Pass through `timeframe`, `period_return_amount`, `period_return_pct`, `as_of`
   - Series range from `points[0].date` / `points[last].date` (no `window_start` / `window_end` in API response)
   - Omit `account_ids` from this insight's payload

**Notes:**

- All calculation rules (carry-forward, return type) are defined in [performance-history API](../../../design-api/examples/net-worth/net-worth-apis.md#get-v1performance-history) and [net worth core](../net-worth/net-worth-core.md).
- Investment accounts are assets — signed rollup equals raw sum of `balances_current`.
- **Not available in current Plaid schema:** benchmark overlay, time-weighted / money-weighted return, per-security performance lines, historical return from `cost_basis`.

### Data output

**Formatting:** Dollar fields — 2 dp; fraction percent fields — 3 dp ([SKILL.md](../../SKILL.md#output-formatting)).

| Field | Type | Description |
|---|---|---|
| `timeframe` | string | Requested window: `trailing_1m`, `trailing_3m`, `ytd`, `trailing_1y`, `all_time` |
| `points` | array | Chart series: `{ date, total_value }`, sorted ascending by `date` |
| `start_value` | number | `points[0].total_value` |
| `end_value` | number | `points[last].total_value` |
| `period_return_amount` | number | `end_value - start_value` |
| `period_return_pct` | number | Fraction (e.g. `0.035` = 3.5%) — 3 dp; omit if `start_value` ≤ 0 |
| `as_of` | date | Last point date |
