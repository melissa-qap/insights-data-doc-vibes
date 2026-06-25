# API spec examples

Reference API surface docs following the [design-api SKILL](../SKILL.md) output template.

**Insight calculations:** Logic lives in [outline-plaid-insight](../../outline-plaid-insight/examples/README.md). API specs link to core partials and insight specs — do not duplicate calculation SQL here.

**To add a new domain:** save `examples/{domain}/{domain}-apis.md`, then add a row below and to the calibration table in [SKILL.md](../SKILL.md).

## Index

| Domain | File | Endpoints |
|---|---|---|
| Net worth | [net-worth/net-worth-apis.md](net-worth/net-worth-apis.md) | `POST /v1/plaid/sync/balances`, `GET /v1/accounts`, `GET /v1/performance-history`, `GET /v1/account-balance`, `GET /v1/assets-liabilities` |
| Investment account | [investment-account/investment-account-apis.md](investment-account/investment-account-apis.md) | `POST /v1/plaid/sync/balances` (cross-ref), `POST /v1/plaid/sync/investment-holdings` |
| Cash flow | [cash-flow/cash-flow-apis.md](cash-flow/cash-flow-apis.md) | `POST /v1/plaid/sync/transactions`, `GET /v1/cash-flow/recurrences` |
