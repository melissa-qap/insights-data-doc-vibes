# Plaid API Response Schema — Reference

Hybrid reference: **datatable mappings** for insight queries plus **API field documentation** mirroring [Plaid's official API spec](https://plaid.com/docs/api/). Insight logic queries datatables only — never call Plaid APIs at runtime.

**Assumption:** All Plaid API responses are already imported into the datatables below.

---

## Plaid API coverage

**Linked:** Yes = endpoint response is imported into a datatable defined in this file. Insight logic uses datatables only — never calls Plaid APIs at runtime. See [Table index](#table-index) for datatable names.

| Category | Endpoint | Description | Linked |
|---|---|---|---|
| Accounts & balance | `/accounts/get` | List all active Item accounts with cached balances | Yes |
| Accounts & balance | `/accounts/balance/get` | Real-time account balances via live institution call | Yes |
| Transactions | `/transactions/sync` | Incremental transaction updates via cursor (recommended) | Yes |
| Transactions | `/transactions/get` | Paginated transaction fetch (legacy) | No |
| Transactions | `/transactions/recurring/get` | Plaid-detected recurring streams (subscriptions, bills, income) | No |
| Transactions | `/transactions/refresh` | Force on-demand transaction refresh | No |
| Transactions | `/categories/get` | Full Plaid transaction category taxonomy | No |
| Transactions | `/transactions/enrich` | Enrich merchant/category on your own transaction data | No |
| Investments | `/investments/holdings/get` | Holdings, securities, and investment account balances | Yes |
| Investments | `/investments/transactions/get` | Buy/sell/dividend/fee investment transactions | No |
| Investments | `/investments/refresh` | Force on-demand investment data refresh | No |
| Liabilities | `/liabilities/get` | Credit card, student loan, and mortgage liability details | Yes |
| Signal & Balance | `/signal/evaluate` | Score ACH return risk for a proposed debit | No |
| Signal & Balance | `/signal/decision/report` | Report whether a scored ACH debit was initiated | No |
| Signal & Balance | `/signal/return/report` | Report an ACH return on a scored transaction | No |
| Signal & Balance | `/signal/prepare` | Enable an Item for Signal Transaction Scores | No |
| Transfer | `/transfer/authorization/create` | Pre-authorize a transfer | No |
| Transfer | `/transfer/authorization/cancel` | Cancel a transfer authorization | No |
| Transfer | `/transfer/create` | Initiate a transfer | No |
| Transfer | `/transfer/cancel` | Cancel a pending transfer | No |
| Transfer | `/transfer/get` | Get transfer details | No |
| Transfer | `/transfer/list` | List transfers | No |
| Transfer | `/transfer/event/list` | List transfer lifecycle events | No |
| Transfer | `/transfer/event/sync` | Sync transfer events incrementally | No |
| Transfer | `/transfer/sweep/get` | Get sweep (funding) details | No |
| Transfer | `/transfer/sweep/list` | List sweeps | No |
| Transfer | `/transfer/capabilities/get` | Check RTP eligibility for an Item | No |
| Transfer | `/transfer/intent/create` | Create transfer intent for Transfer UI | No |
| Transfer | `/transfer/intent/get` | Get transfer intent | No |
| Transfer | `/transfer/migrate_account` | Create Item from known account/routing numbers | No |
| Transfer | `/transfer/recurring/create` | Create recurring transfer | No |
| Transfer | `/transfer/recurring/cancel` | Cancel recurring transfer | No |
| Transfer | `/transfer/recurring/get` | Get recurring transfer | No |
| Transfer | `/transfer/recurring/list` | List recurring transfers | No |
| Transfer | `/transfer/refund/create` | Refund a transfer | No |
| Transfer | `/transfer/refund/cancel` | Cancel a refund | No |
| Transfer | `/transfer/refund/get` | Get refund details | No |
| Transfer | `/transfer/platform/originator/create` | Onboard transfer originator (Platforms) | No |
| Transfer | `/transfer/platform/person/create` | Create beneficial owner or control person | No |
| Transfer | `/transfer/platform/requirement/submit` | Submit onboarding requirements | No |
| Transfer | `/transfer/platform/document/submit` | Submit onboarding documents | No |
| Transfer | `/transfer/originator/get` | Get originator onboarding status | No |
| Transfer | `/transfer/originator/list` | List originator statuses | No |
| Transfer | `/transfer/originator/funding_account/create` | Add originator funding account | No |
| Transfer | `/transfer/ledger/deposit` | Deposit into Plaid ledger | No |
| Transfer | `/transfer/ledger/distribute` | Move balance between platform and originator | No |
| Transfer | `/transfer/ledger/get` | Get ledger balance | No |
| Transfer | `/transfer/ledger/withdraw` | Withdraw from ledger | No |
| Transfer | `/transfer/ledger/event/list` | List ledger events | No |
| Transfer | `/transfer/metrics/get` | Transfer product usage metrics | No |
| Transfer | `/transfer/configuration/get` | Transfer product configuration | No |
| Payment Initiation | `/payment_initiation/recipient/create` | Create payment recipient | No |
| Payment Initiation | `/payment_initiation/recipient/get` | Get payment recipient | No |
| Payment Initiation | `/payment_initiation/recipient/list` | List payment recipients | No |
| Payment Initiation | `/payment_initiation/payment/create` | Create payment or standing order | No |
| Payment Initiation | `/payment_initiation/payment/get` | Get payment status | No |
| Payment Initiation | `/payment_initiation/payment/list` | List payments | No |
| Payment Initiation | `/payment_initiation/payment/reverse` | Refund from virtual account | No |
| Payment Initiation | `/payment_initiation/consent/create` | Create payment consent | No |
| Payment Initiation | `/payment_initiation/consent/get` | Get payment consent | No |
| Payment Initiation | `/payment_initiation/consent/revoke` | Revoke payment consent | No |
| Payment Initiation | `/payment_initiation/consent/payment/execute` | Execute payment using consent | No |
| Payment Initiation | `/wallet/create` | Create Plaid virtual account | No |
| Payment Initiation | `/wallet/get` | Get virtual account | No |
| Payment Initiation | `/wallet/list` | List virtual accounts | No |
| Payment Initiation | `/wallet/transaction/execute` | Execute wallet transaction | No |
| Payment Initiation | `/wallet/transaction/get` | Get wallet transaction | No |
| Payment Initiation | `/wallet/transaction/list` | List wallet transactions | No |

`/accounts/balance/get` is listed under Accounts & balance only; Plaid also documents it under Signal & Balance.

---

## Account APIs overview

Plaid uses a shared **Account object** across endpoints. There is no single "get everything" endpoint — account metadata, balances, and enrichment come from different products.

### Endpoint comparison

| Endpoint | Plaid docs | Account scope | Balance freshness | Extra account fields | Maps to datatable |
|---|---|---|---|---|---|
| `/accounts/get` | [Accounts API](https://plaid.com/docs/api/accounts/#accountsget) | All active Item accounts | Cached (~daily if Transactions/Investments/Liabilities enabled) | Standard Account object | `plaid_accounts` (alternate) |
| `/accounts/balance/get` | [Signal/Balance API](https://plaid.com/docs/api/products/signal/#accountsbalanceget) | All active Item accounts | Real-time (sync call to institution) | Standard Account object | `plaid_accounts` (primary) |
| `/liabilities/get` | [Liabilities API](https://plaid.com/docs/api/products/liabilities/) | All Item accounts | Cached (~daily) | Standard Account object + `liabilities.*` | Liability tables; `accounts[]` joinable to `plaid_accounts` |
| `/investments/holdings/get` | [Investments API](https://plaid.com/docs/api/products/investments/) | Investment-type accounts only | Cached | `balances.margin_loan_amount` | Holdings/securities tables; investment `accounts[]` |
| `/auth/get` | [Auth API](https://plaid.com/docs/api/products/auth/) | Checking, savings, cash management | Cached when present | `verification_status`, `persistent_account_id` + `numbers.*` | *(secondary — no datatable)* |
| `/identity/get` | [Identity API](https://plaid.com/docs/api/products/identity/) | Filterable by `account_ids` | N/A | `owners[]` (names, emails, phones, addresses) | *(secondary — no datatable)* |
| `/transactions/sync` | [Transactions API](https://plaid.com/docs/api/products/transactions/) | Subset: accounts with transactions in response | Cached | Standard Account object | `plaid_transactions` only; **not** canonical account list |
| `/transactions/get` | [Transactions API](https://plaid.com/docs/api/products/transactions/) | Same subset as sync | Cached | Standard Account object | *(secondary — prefer sync)* |

**Out of scope:** `/signal/evaluate`, `/processor/*` variants.

### Import guidance for `plaid_accounts`

| Rule | Detail |
|---|---|
| **Primary source** | `/accounts/balance/get` — real-time balances for PFM snapshots |
| **Alternate source** | `/accounts/get` — free, cached balances; acceptable for lower-cost refresh |
| **Do not use** | `/transactions/sync` or `/transactions/get` `accounts[]` as the canonical account list — partial subset; omits investment accounts |
| **Investment margin** | Populate `balances_margin_loan_amount` from `/investments/holdings/get` `accounts[]` when available (investment accounts only) |
| **Insight fields used** | `account_id`, `name`, `mask`, `type`, `subtype`, `balances_current`, `balances_available`, `item_id` |

---

## Data Tables

Each endpoint maps to one or more tables. Nested response fields are flattened with underscores (e.g. `balances.current` → `balances_current`).

### Common columns (all tables)

| Column | Type | Notes |
|---|---|---|
| `user_id` | string | User scope — filter all queries by this |
| `item_id` | string | Plaid Item that produced this row |
| `synced_at` | timestamp | When this row was imported from Plaid |

For tables that accumulate history across syncs, keep one row per entity **per sync** (do not overwrite prior imports). Use `synced_at` to select current vs historical snapshots.

### Table index

See [Plaid API coverage](#plaid-api-coverage) for the full product-area endpoint list and linked status.

| Endpoint | Table(s) |
|---|---|
| `/accounts/balance/get` (primary), `/accounts/get` (alternate) | `plaid_accounts` |
| `/transactions/sync` | `plaid_transactions` |
| `/investments/holdings/get` | `plaid_investment_holdings`, `plaid_investment_securities` |
| `/liabilities/get` | `plaid_liabilities_credit`, `plaid_liabilities_student`, `plaid_liabilities_mortgage` |
| *(extension)* | `plaid_items` |

---

## Extension tables

Tables maintained outside Plaid imports. Used when Plaid fields are insufficient for an insight.

### `plaid_items`

Source: Plaid Item metadata at link time (e.g. from `/link/token/create` callback or Items API). One row per Item per user.

| Column | Type | Notes |
|---|---|---|
| `item_id` | string | Join to `item_id` on all Plaid datatables |
| `institution_name` | string | Display name for the linked institution |
| `institution_id` | string \| null | Optional Plaid institution ID (e.g. `ins_3`) |

Used by [investment accounts by institution](examples/investment-account/investment-accounts-by-institution.md) to group accounts under institution labels. `item_id` alone is on every datatable row; institution names are not.

---

### `plaid_accounts`

Source: `/accounts/balance/get` (primary) or `/accounts/get` (alternate) → `accounts[]`. One row per account per sync. Both endpoints produce identical column mapping.

Optional enrichment: `balances_margin_loan_amount` from `/investments/holdings/get` `accounts[]` (investment accounts only; nullable otherwise).

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `name` | string | `name` |
| `official_name` | string \| null | `official_name` |
| `mask` | string \| null | `mask` |
| `type` | string | See [enum_account_type](#enum_account_type) |
| `subtype` | string \| null | See [enum_account_subtype](#enum_account_subtype) |
| `balances_available` | number \| null | `balances.available` |
| `balances_current` | number \| null | `balances.current` |
| `balances_limit` | number \| null | `balances.limit` |
| `balances_iso_currency_code` | string \| null | `balances.iso_currency_code` |
| `balances_unofficial_currency_code` | string \| null | `balances.unofficial_currency_code` |
| `balances_last_updated_datetime` | string \| null | `balances.last_updated_datetime` |
| `balances_margin_loan_amount` | number \| null | `balances.margin_loan_amount` (investment endpoints only) |

**Query patterns:**
- Current state: `WHERE user_id = ? AND synced_at = (SELECT MAX(synced_at) FROM plaid_accounts WHERE user_id = ?)`
- Point-in-time: `WHERE user_id = ? AND synced_at <= ?` then take latest row per `account_id`

---

### `plaid_transactions`

Source: `/transactions/sync` → `added[]`, `modified[]`, `removed[]`. One row per transaction (upsert on `transaction_id`; retain latest `synced_at`). Rows from `removed[]` set `removed = true` (row retained, not deleted).

| Column | Type | Source field |
|---|---|---|
| `transaction_id` | string | `transaction_id` |
| `account_id` | string | `account_id` |
| `amount` | number | `amount` |
| `date` | date | `date` |
| `authorized_date` | date \| null | `authorized_date` |
| `name` | string | `name` |
| `merchant_name` | string \| null | `merchant_name` |
| `pending` | boolean | `pending` |
| `removed` | boolean | Set `true` when transaction appears in `removed[]`; default `false` for `added[]` / `modified[]` |
| `payment_channel` | string | See [enum_payment_channel](#enum_payment_channel) |
| `personal_finance_category_primary` | string | See [enum_pfc_primary](#enum_pfc_primary) |
| `personal_finance_category_detailed` | string | See [enum_pfc_detailed](#enum_pfc_detailed) |
| `personal_finance_category_confidence_level` | string | See [enum_pfc_confidence_level](#enum_pfc_confidence_level) |
| `location_city` | string \| null | `location.city` |
| `location_region` | string \| null | `location.region` |
| `iso_currency_code` | string \| null | `iso_currency_code` |
| `unofficial_currency_code` | string \| null | `unofficial_currency_code` |

---

### `plaid_investment_holdings`

Source: `/investments/holdings/get` → `holdings[]`. One row per holding per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `security_id` | string | `security_id` |
| `quantity` | number | `quantity` |
| `cost_basis` | number \| null | `cost_basis` |
| `institution_value` | number \| null | `institution_value` |
| `institution_price` | number \| null | `institution_price` |
| `institution_price_as_of` | date \| null | `institution_price_as_of` |
| `iso_currency_code` | string \| null | `iso_currency_code` |
| `unofficial_currency_code` | string \| null | `unofficial_currency_code` |

---

### `plaid_investment_securities`

Source: `/investments/holdings/get` → `securities[]`. One row per security per sync.

| Column | Type | Source field |
|---|---|---|
| `security_id` | string | `security_id` |
| `name` | string \| null | `name` |
| `ticker_symbol` | string \| null | `ticker_symbol` |
| `type` | string | Summary security type — see [enum_investment_security_type](#enum_investment_security_type); normalize via [enum_investment_security_subtype](#enum_investment_security_subtype) when raw value is not a summary type |
| `isin` | string \| null | `isin` |
| `close_price` | number \| null | `close_price` |
| `close_price_as_of` | date \| null | `close_price_as_of` |
| `iso_currency_code` | string \| null | `iso_currency_code` |

Join to `plaid_investment_holdings` on `security_id` + matching `synced_at`.

---

### `plaid_liabilities_credit`

Source: `/liabilities/get` → `liabilities.credit[]`. One row per credit account per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `aprs` | JSON | `aprs[]` |
| `is_overdue` | boolean \| null | `is_overdue` |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `last_statement_issue_date` | date \| null | `last_statement_issue_date` |
| `last_statement_balance` | number \| null | `last_statement_balance` |
| `minimum_payment_amount` | number \| null | `minimum_payment_amount` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |

---

### `plaid_liabilities_student`

Source: `/liabilities/get` → `liabilities.student[]`. One row per student loan per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `interest_rate_percentage` | number | `interest_rate_percentage` |
| `is_overdue` | boolean \| null | `is_overdue` |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `loan_name` | string \| null | `loan_name` |
| `loan_status_type` | string | See [enum_loan_status_type](#enum_loan_status_type) |
| `minimum_payment_amount` | number \| null | `minimum_payment_amount` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |
| `origination_date` | date \| null | `origination_date` |
| `origination_principal_amount` | number \| null | `origination_principal_amount` |
| `outstanding_interest_amount` | number \| null | `outstanding_interest_amount` |

---

### `plaid_liabilities_mortgage`

Source: `/liabilities/get` → `liabilities.mortgage[]`. One row per mortgage per sync.

| Column | Type | Source field |
|---|---|---|
| `account_id` | string | `account_id` |
| `interest_rate_percentage` | number \| null | `interest_rate.percentage` |
| `interest_rate_type` | string \| null | See [enum_interest_rate_type](#enum_interest_rate_type) |
| `last_payment_amount` | number \| null | `last_payment_amount` |
| `last_payment_date` | date \| null | `last_payment_date` |
| `loan_term` | string \| null | `loan_term` |
| `minimum_monthly_payment` | number \| null | `minimum_monthly_payment` |
| `next_payment_due_date` | date \| null | `next_payment_due_date` |
| `origination_date` | date \| null | `origination_date` |
| `origination_principal_amount` | number \| null | `origination_principal_amount` |
| `past_due_amount` | number \| null | `past_due_amount` |
| `ytd_interest_paid` | number \| null | `ytd_interest_paid` |
| `ytd_principal_paid` | number \| null | `ytd_principal_paid` |

---

## Enum reference

Lookup tables for enum-like datatable columns. One row per allowed value.

### Table formats

| Enum shape | Columns | Used for |
| --- | --- | --- |
| Flat | `Value` \| `Description` | Single-level enums (account type, payment channel, etc.) |
| Hierarchical | `Type` \| `Subtype` \| `Description` | Parent/child enums — one row per subtype |

### Enum index

| Datatable column | Enum table |
| --- | --- |
| `plaid_accounts.type` | [enum_account_type](#enum_account_type) |
| `plaid_accounts.subtype` | [enum_account_subtype](#enum_account_subtype) |
| `plaid_transactions.payment_channel` | [enum_payment_channel](#enum_payment_channel) |
| `plaid_transactions.personal_finance_category_primary` | [enum_pfc_primary](#enum_pfc_primary) |
| `plaid_transactions.personal_finance_category_detailed` | [enum_pfc_detailed](#enum_pfc_detailed) |
| `plaid_transactions.personal_finance_category_confidence_level` | [enum_pfc_confidence_level](#enum_pfc_confidence_level) |
| `plaid_investment_securities.type` | [enum_investment_security_type](#enum_investment_security_type) |
| *(API only)* `securities[].subtype` | [enum_investment_security_subtype](#enum_investment_security_subtype) |
| `plaid_liabilities_student.loan_status_type` | [enum_loan_status_type](#enum_loan_status_type) |
| `plaid_liabilities_mortgage.interest_rate_type` | [enum_interest_rate_type](#enum_interest_rate_type) |

---

### enum_account_type

`plaid_accounts.type` — high-level account classification.

| Value | Description |
| --- | --- |
| `investment` | Investment account (brokerage, 401k, IRA, etc.) |
| `credit` | Credit card or line of credit |
| `depository` | Cash account (checking, savings, etc.) |
| `loan` | Loan account (mortgage, student, auto, etc.) |
| `brokerage` | Legacy type name for investment (API ≤ 2018-05-22) |
| `other` | Non-specified account type |

Source: [Plaid Account type schema](https://plaid.com/docs/api/accounts/#account-type-schema)

---

### enum_account_subtype

`plaid_accounts.subtype` — account subtype. Valid combinations depend on `type`.

| Type | Subtype | Description |
| --- | --- | --- |
| `depository` | `checking` | Checking account |
| `depository` | `savings` | Savings account |
| `depository` | `hsa` | Health Savings Account (cash only, US) |
| `depository` | `cd` | Certificate of deposit |
| `depository` | `money market` | Money market account |
| `depository` | `paypal` | PayPal depository account |
| `depository` | `prepaid` | Prepaid debit card |
| `depository` | `cash management` | Cash management account at a brokerage |
| `depository` | `ebt` | Electronic Benefit Transfer (US) |
| `depository` | `cash isa` | Cash ISA (UK) |
| `depository` | `business` | Business depository account |
| `depository` | `payroll` | Payroll account |
| `depository` | `limited purpose checking` | Limited-purpose checking (opt-in in Link) |
| `credit` | `credit card` | Bank-issued credit card |
| `credit` | `paypal` | PayPal-issued credit card |
| `credit` | `line of credit` | Pre-approved line of credit |
| `loan` | `auto` | Auto loan |
| `loan` | `business` | Business loan |
| `loan` | `commercial` | Commercial loan |
| `loan` | `construction` | Construction loan |
| `loan` | `consumer` | Consumer loan |
| `loan` | `home equity` | Home Equity Line of Credit (HELOC) |
| `loan` | `loan` | General loan |
| `loan` | `mortgage` | Mortgage loan |
| `loan` | `overdraft` | Pre-approved overdraft tied to checking |
| `loan` | `line of credit` | Pre-approved line of credit |
| `loan` | `student` | Student loan |
| `loan` | `other` | Other or unknown loan type |
| `investment` | `401a` | 401(a) retirement plan (US) |
| `investment` | `401k` | 401(k) retirement account (US) |
| `investment` | `403B` | 403(b) retirement account (US) |
| `investment` | `457b` | 457(b) deferred-compensation plan (US) |
| `investment` | `529` | 529 college savings plan (US) |
| `investment` | `brokerage` | Standard brokerage account |
| `investment` | `cash isa` | Interest-bearing ISA (UK) |
| `investment` | `crypto exchange` | Cryptocurrency exchange account |
| `investment` | `education savings account` | Coverdell ESA (US) |
| `investment` | `fhsa` | First Home Savings Account (Canada) |
| `investment` | `fixed annuity` | Fixed annuity |
| `investment` | `gic` | Guaranteed Investment Certificate (Canada) |
| `investment` | `health reimbursement arrangement` | Health Reimbursement Arrangement (US) |
| `investment` | `hsa` | Health Savings Account (non-cash, US) |
| `investment` | `ira` | Traditional IRA (US) |
| `investment` | `isa` | Individual Savings Account (UK) |
| `investment` | `keogh` | Keogh self-employed retirement plan (US) |
| `investment` | `lif` | Life Income Fund (Canada) |
| `investment` | `life insurance` | Life insurance account |
| `investment` | `lira` | Locked-in Retirement Account (Canada) |
| `investment` | `lrif` | Locked-in Retirement Income Fund (Canada) |
| `investment` | `lrsp` | Locked-in Retirement Savings Plan (Canada) |
| `investment` | `mutual fund` | Mutual fund account |
| `investment` | `non-custodial wallet` | Non-custodial crypto wallet |
| `investment` | `non-taxable brokerage account` | Non-taxable brokerage account |
| `investment` | `other` | Other investment account |
| `investment` | `other annuity` | Other annuity account |
| `investment` | `other insurance` | Other insurance account |
| `investment` | `pension` | Standard pension account |
| `investment` | `prif` | Prescribed Registered Retirement Income Fund (Canada) |
| `investment` | `profit sharing plan` | Profit sharing plan |
| `investment` | `qshr` | Qualifying share account |
| `investment` | `rdsp` | Registered Disability Savings Plan (Canada) |
| `investment` | `resp` | Registered Education Savings Plan (Canada) |
| `investment` | `retirement` | Retirement account (other) |
| `investment` | `rlif` | Restricted Life Income Fund (Canada) |
| `investment` | `roth` | Roth IRA (US) |
| `investment` | `roth 401k` | Roth 401(k) (US) |
| `investment` | `roth 403B` | Roth 403(b) (US) |
| `investment` | `roth 457b` | Roth 457(b) (US) |
| `investment` | `roth pension` | Roth pension account |
| `investment` | `roth profit sharing plan` | Roth profit sharing plan |
| `investment` | `roth thrift savings plan` | Roth Thrift Savings Plan (US) |
| `investment` | `rrif` | Registered Retirement Income Fund (Canada) |
| `investment` | `rrsp` | Registered Retirement Savings Plan (Canada) |
| `investment` | `sarsep` | SARSEP retirement plan (US) |
| `investment` | `sep ira` | SEP IRA (US) |
| `investment` | `simple ira` | SIMPLE IRA (US) |
| `investment` | `sipp` | Self-Invested Personal Pension (UK) |
| `investment` | `stock plan` | Stock plan account |
| `investment` | `thrift savings plan` | Thrift Savings Plan (US) |
| `investment` | `tfsa` | Tax-Free Savings Account (Canada) |
| `investment` | `trust` | Trust account |
| `investment` | `ugma` | Uniform Gift to Minors Act (US) |
| `investment` | `utma` | Uniform Transfers to Minors Act (US) |
| `investment` | `variable annuity` | Variable annuity |
| `other` | `other` | Other or unknown account type |

Source: [Plaid Account type schema](https://plaid.com/docs/api/accounts/#account-type-schema)

---

### enum_payment_channel

`plaid_transactions.payment_channel` — how the transaction was initiated.

| Value | Description |
| --- | --- |
| `online` | Online or digital payment |
| `in store` | In-person at a physical location |
| `other` | Other or unknown channel |

---

### enum_pfc_primary

`plaid_transactions.personal_finance_category_primary` — top-level transaction category (PFCv1).

| Value | Description |
| --- | --- |
| `INCOME` | Wages, dividends, interest, tax refunds, and other income |
| `TRANSFER_IN` | Inbound transfers, deposits, and loan disbursements |
| `TRANSFER_OUT` | Outbound transfers, withdrawals, and investment funding |
| `LOAN_PAYMENTS` | Credit card, mortgage, student, auto, and other loan payments |
| `BANK_FEES` | ATM, overdraft, interest, and other bank fees |
| `ENTERTAINMENT` | Casinos, music, movies, games, and sporting events |
| `FOOD_AND_DRINK` | Restaurants, groceries, coffee, and fast food |
| `GENERAL_MERCHANDISE` | Retail, clothing, electronics, and online marketplaces |
| `HOME_IMPROVEMENT` | Furniture, hardware, repair, and security |
| `MEDICAL` | Dental, pharmacy, primary care, and veterinary |
| `PERSONAL_CARE` | Gyms, hair and beauty, laundry |
| `GENERAL_SERVICES` | Accounting, automotive, childcare, education, insurance |
| `GOVERNMENT_AND_NON_PROFIT` | Donations, taxes, and government agencies |
| `TRANSPORTATION` | Gas, parking, public transit, taxis, tolls |
| `TRAVEL` | Flights, lodging, rental cars |
| `RENT_AND_UTILITIES` | Rent, gas/electric, internet, water, phone |

PFCv2 adds subcategories (income, loan disbursement/repayment, bank fees). Default import is PFCv1 unless `personal_finance_category_version: v2` is set. See [PFC migration guide](https://plaid.com/docs/transactions/pfc-migration/).

---

### enum_pfc_detailed

`plaid_transactions.personal_finance_category_detailed` — granular transaction category (PFCv1).

| Type | Subtype | Description |
| --- | --- | --- |
| `INCOME` | `INCOME_DIVIDENDS` | Dividends from investment accounts |
| `INCOME` | `INCOME_INTEREST_EARNED` | Income from interest on savings accounts |
| `INCOME` | `INCOME_RETIREMENT_PENSION` | Income from pension payments  |
| `INCOME` | `INCOME_TAX_REFUND` | Income from tax refunds |
| `INCOME` | `INCOME_UNEMPLOYMENT` | Income from unemployment benefits, including unemployment insurance and healthcare |
| `INCOME` | `INCOME_WAGES` | Income from salaries, gig-economy work, and tips earned |
| `INCOME` | `INCOME_OTHER_INCOME` | Other miscellaneous income, including alimony, social security, child support, and rental |
| `TRANSFER_IN` | `TRANSFER_IN_CASH_ADVANCES_AND_LOANS` | Loans and cash advances deposited into a bank account |
| `TRANSFER_IN` | `TRANSFER_IN_DEPOSIT` | Cash, checks, and ATM deposits into a bank account |
| `TRANSFER_IN` | `TRANSFER_IN_INVESTMENT_AND_RETIREMENT_FUNDS` | Inbound transfers to an investment or retirement account |
| `TRANSFER_IN` | `TRANSFER_IN_SAVINGS` | Inbound transfers to a savings account |
| `TRANSFER_IN` | `TRANSFER_IN_ACCOUNT_TRANSFER` | General inbound transfers from another account |
| `TRANSFER_IN` | `TRANSFER_IN_OTHER_TRANSFER_IN` | Other miscellaneous inbound transactions |
| `TRANSFER_OUT` | `TRANSFER_OUT_INVESTMENT_AND_RETIREMENT_FUNDS` | Transfers to an investment or retirement account, including investment apps such as Acorns, Betterment |
| `TRANSFER_OUT` | `TRANSFER_OUT_SAVINGS` | Outbound transfers to savings accounts |
| `TRANSFER_OUT` | `TRANSFER_OUT_WITHDRAWAL` | Withdrawals from a bank account |
| `TRANSFER_OUT` | `TRANSFER_OUT_ACCOUNT_TRANSFER` | General outbound transfers to another account |
| `TRANSFER_OUT` | `TRANSFER_OUT_OTHER_TRANSFER_OUT` | Other miscellaneous outbound transactions |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_CAR_PAYMENT` | Car loans and leases |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_CREDIT_CARD_PAYMENT` | Payments to a credit card. These are positive amounts for credit card subtypes and negative for depository subtypes |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_PERSONAL_LOAN_PAYMENT` | Personal loans, including cash advances and buy now pay later repayments |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_MORTGAGE_PAYMENT` | Payments on mortgages |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_STUDENT_LOAN_PAYMENT` | Payments on student loans. For college tuition, refer to "General Services - Education" |
| `LOAN_PAYMENTS` | `LOAN_PAYMENTS_OTHER_PAYMENT` | Other miscellaneous debt payments |
| `BANK_FEES` | `BANK_FEES_ATM_FEES` | Fees incurred for out-of-network ATMs |
| `BANK_FEES` | `BANK_FEES_FOREIGN_TRANSACTION_FEES` | Fees incurred on non-domestic transactions |
| `BANK_FEES` | `BANK_FEES_INSUFFICIENT_FUNDS` | Fees relating to insufficient funds |
| `BANK_FEES` | `BANK_FEES_INTEREST_CHARGE` | Fees incurred for interest on purchases, including not-paid-in-full or interest on cash advances |
| `BANK_FEES` | `BANK_FEES_OVERDRAFT_FEES` | Fees incurred when an account is in overdraft |
| `BANK_FEES` | `BANK_FEES_OTHER_BANK_FEES` | Other miscellaneous bank fees |
| `ENTERTAINMENT` | `ENTERTAINMENT_CASINOS_AND_GAMBLING` | Gambling, casinos, and sports betting |
| `ENTERTAINMENT` | `ENTERTAINMENT_MUSIC_AND_AUDIO` | Digital and in-person music purchases, including music streaming services |
| `ENTERTAINMENT` | `ENTERTAINMENT_SPORTING_EVENTS_AMUSEMENT_PARKS_AND_MUSEUMS` | Purchases made at sporting events, music venues, concerts, museums, and amusement parks |
| `ENTERTAINMENT` | `ENTERTAINMENT_TV_AND_MOVIES` | In home movie streaming services and movie theaters |
| `ENTERTAINMENT` | `ENTERTAINMENT_VIDEO_GAMES` | Digital and in-person video game purchases |
| `ENTERTAINMENT` | `ENTERTAINMENT_OTHER_ENTERTAINMENT` | Other miscellaneous entertainment purchases, including night life and adult entertainment |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_BEER_WINE_AND_LIQUOR` | Beer, Wine & Liquor Stores |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_COFFEE` | Purchases at coffee shops or cafes |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_FAST_FOOD` | Dining expenses for fast food chains |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_GROCERIES` | Purchases for fresh produce and groceries, including farmers' markets |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_RESTAURANT` | Dining expenses for restaurants, bars, gastropubs, and diners |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_VENDING_MACHINES` | Purchases made at vending machine operators |
| `FOOD_AND_DRINK` | `FOOD_AND_DRINK_OTHER_FOOD_AND_DRINK` | Other miscellaneous food and drink, including desserts, juice bars, and delis |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_BOOKSTORES_AND_NEWSSTANDS` | Books, magazines, and news  |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_CLOTHING_AND_ACCESSORIES` | Apparel, shoes, and jewelry |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_CONVENIENCE_STORES` | Purchases at convenience stores |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_DEPARTMENT_STORES` | Retail stores with wide ranges of consumer goods, typically specializing in clothing and home goods |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_DISCOUNT_STORES` | Stores selling goods at a discounted price |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_ELECTRONICS` | Electronics stores and websites |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_GIFTS_AND_NOVELTIES` | Photo, gifts, cards, and floral stores |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_OFFICE_SUPPLIES` | Stores that specialize in office goods |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_ONLINE_MARKETPLACES` | Multi-purpose e-commerce platforms such as Etsy, Ebay and Amazon |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_PET_SUPPLIES` | Pet supplies and pet food |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_SPORTING_GOODS` | Sporting goods, camping gear, and outdoor equipment |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_SUPERSTORES` | Superstores such as Target and Walmart, selling both groceries and general merchandise |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_TOBACCO_AND_VAPE` | Purchases for tobacco and vaping products |
| `GENERAL_MERCHANDISE` | `GENERAL_MERCHANDISE_OTHER_GENERAL_MERCHANDISE` | Other miscellaneous merchandise, including toys, hobbies, and arts and crafts |
| `HOME_IMPROVEMENT` | `HOME_IMPROVEMENT_FURNITURE` | Furniture, bedding, and home accessories |
| `HOME_IMPROVEMENT` | `HOME_IMPROVEMENT_HARDWARE` | Building materials, hardware stores, paint, and wallpaper |
| `HOME_IMPROVEMENT` | `HOME_IMPROVEMENT_REPAIR_AND_MAINTENANCE` | Plumbing, lighting, gardening, and roofing |
| `HOME_IMPROVEMENT` | `HOME_IMPROVEMENT_SECURITY` | Home security system purchases |
| `HOME_IMPROVEMENT` | `HOME_IMPROVEMENT_OTHER_HOME_IMPROVEMENT` | Other miscellaneous home purchases, including pool installation and pest control |
| `MEDICAL` | `MEDICAL_DENTAL_CARE` | Dentists and general dental care |
| `MEDICAL` | `MEDICAL_EYE_CARE` | Optometrists, contacts, and glasses stores |
| `MEDICAL` | `MEDICAL_NURSING_CARE` | Nursing care and facilities |
| `MEDICAL` | `MEDICAL_PHARMACIES_AND_SUPPLEMENTS` | Pharmacies and nutrition shops |
| `MEDICAL` | `MEDICAL_PRIMARY_CARE` | Doctors and physicians |
| `MEDICAL` | `MEDICAL_VETERINARY_SERVICES` | Prevention and care procedures for animals |
| `MEDICAL` | `MEDICAL_OTHER_MEDICAL` | Other miscellaneous medical, including blood work, hospitals, and ambulances |
| `PERSONAL_CARE` | `PERSONAL_CARE_GYMS_AND_FITNESS_CENTERS` | Gyms, fitness centers, and workout classes |
| `PERSONAL_CARE` | `PERSONAL_CARE_HAIR_AND_BEAUTY` | Manicures, haircuts, waxing, spa/massages, and bath and beauty products  |
| `PERSONAL_CARE` | `PERSONAL_CARE_LAUNDRY_AND_DRY_CLEANING` | Wash and fold, and dry cleaning expenses |
| `PERSONAL_CARE` | `PERSONAL_CARE_OTHER_PERSONAL_CARE` | Other miscellaneous personal care, including mental health apps and services |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_ACCOUNTING_AND_FINANCIAL_PLANNING` | Financial planning, and tax and accounting services |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_AUTOMOTIVE` | Oil changes, car washes, repairs, and towing |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_CHILDCARE` | Babysitters and daycare |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_CONSULTING_AND_LEGAL` | Consulting and legal services |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_EDUCATION` | Elementary, high school, professional schools, and college tuition |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_INSURANCE` | Insurance for auto, home, and healthcare |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_POSTAGE_AND_SHIPPING` | Mail, packaging, and shipping services |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_STORAGE` | Storage services and facilities |
| `GENERAL_SERVICES` | `GENERAL_SERVICES_OTHER_GENERAL_SERVICES` | Other miscellaneous services, including advertising and cloud storage  |
| `GOVERNMENT_AND_NON_PROFIT` | `GOVERNMENT_AND_NON_PROFIT_DONATIONS` | Charitable, political, and religious donations |
| `GOVERNMENT_AND_NON_PROFIT` | `GOVERNMENT_AND_NON_PROFIT_GOVERNMENT_DEPARTMENTS_AND_AGENCIES` | Government departments and agencies, such as driving licences, and passport renewal |
| `GOVERNMENT_AND_NON_PROFIT` | `GOVERNMENT_AND_NON_PROFIT_TAX_PAYMENT` | Tax payments, including income and property taxes |
| `GOVERNMENT_AND_NON_PROFIT` | `GOVERNMENT_AND_NON_PROFIT_OTHER_GOVERNMENT_AND_NON_PROFIT` | Other miscellaneous government and non-profit agencies |
| `TRANSPORTATION` | `TRANSPORTATION_BIKES_AND_SCOOTERS` | Bike and scooter rentals |
| `TRANSPORTATION` | `TRANSPORTATION_GAS` | Purchases at a gas station |
| `TRANSPORTATION` | `TRANSPORTATION_PARKING` | Parking fees and expenses |
| `TRANSPORTATION` | `TRANSPORTATION_PUBLIC_TRANSIT` | Public transportation, including rail and train, buses, and metro |
| `TRANSPORTATION` | `TRANSPORTATION_TAXIS_AND_RIDE_SHARES` | Taxi and ride share services |
| `TRANSPORTATION` | `TRANSPORTATION_TOLLS` | Toll expenses |
| `TRANSPORTATION` | `TRANSPORTATION_OTHER_TRANSPORTATION` | Other miscellaneous transportation expenses |
| `TRAVEL` | `TRAVEL_FLIGHTS` | Airline expenses |
| `TRAVEL` | `TRAVEL_LODGING` | Hotels, motels, and hosted accommodation such as Airbnb |
| `TRAVEL` | `TRAVEL_RENTAL_CARS` | Rental cars, charter buses, and trucks |
| `TRAVEL` | `TRAVEL_OTHER_TRAVEL` | Other miscellaneous travel expenses |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_GAS_AND_ELECTRICITY` | Gas and electricity bills |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_INTERNET_AND_CABLE` | Internet and cable bills |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_RENT` | Rent payment |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_SEWAGE_AND_WASTE_MANAGEMENT` | Sewage and garbage disposal bills |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_TELEPHONE` | Cell phone bills |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_WATER` | Water bills |
| `RENT_AND_UTILITIES` | `RENT_AND_UTILITIES_OTHER_UTILITIES` | Other miscellaneous utility bills |

Source: [Plaid PFCv1 taxonomy CSV](https://plaid.com/documents/transactions-personal-finance-category-taxonomy.csv). PFCv2-only detailed values are not listed — see [PFC migration guide](https://plaid.com/docs/transactions/pfc-migration/).

---

### enum_pfc_confidence_level

`plaid_transactions.personal_finance_category_confidence_level` — categorization confidence.

| Value | Description |
| --- | --- |
| `VERY_HIGH` | Very high confidence |
| `HIGH` | High confidence |
| `MEDIUM` | Medium confidence |
| `LOW` | Low confidence |
| `UNKNOWN` | Confidence unknown |

---

### enum_investment_security_type

`plaid_investment_securities.type` — Plaid summary security type (`securities[].type`). These 9 values are what the datatable normally stores. These types are also the allocation output buckets for [asset allocation](examples/investment-account/asset-allocation.md) and [investment account detail](examples/investment-account/investment-account-detail.md).

| Value | Description |
| --- | --- |
| `cash` | Cash, currency, and money market funds |
| `cryptocurrency` | Digital or virtual currencies |
| `derivative` | Options, warrants, and other derivative instruments |
| `equity` | Domestic and foreign equities |
| `etf` | Multi-asset exchange-traded investment funds |
| `fixed income` | Bonds and certificates of deposit |
| `loan` | Loans and loan receivables |
| `mutual fund` | Open- and closed-end pooled vehicles |
| `other` | Unknown or other investment types |

Source: [Plaid Investments API](https://plaid.com/docs/api/products/investments/)

---

### enum_investment_security_subtype

`securities[].subtype` — granular security classification (API only; not in datatable today). Maps raw subtype values to summary types in [enum_investment_security_type](#enum_investment_security_type).

| Type | Subtype | Description |
| --- | --- | --- |
| `cash` | `cash` | Cash or currency holding |
| `cash` | `cash management bill` | Cash management bill |
| `cash` | `money market debt` | Money market debt instrument |
| `cryptocurrency` | `cryptocurrency` | Cryptocurrency |
| `derivative` | `derivative` | Derivative instrument |
| `derivative` | `option` | Option contract |
| `derivative` | `warrant` | Warrant |
| `equity` | `equity` | Equity security |
| `equity` | `common stock` | Common stock |
| `equity` | `depositary receipt` | Depositary receipt |
| `equity` | `preferred equity` | Preferred equity |
| `equity` | `convertible equity` | Convertible equity |
| `equity` | `structured equity product` | Structured equity product |
| `equity` | `unit` | Unit |
| `equity` | `real estate investment trust` | Real estate investment trust |
| `equity` | `limited partnership unit` | Limited partnership unit |
| `etf` | `etf` | Exchange-traded fund |
| `fixed income` | `fixed income` | Fixed income security |
| `fixed income` | `bond` | Bond |
| `fixed income` | `municipal bond` | Municipal bond |
| `fixed income` | `convertible bond` | Convertible bond |
| `fixed income` | `mortgage backed security` | Mortgage-backed security |
| `fixed income` | `treasury inflation protected securities` | Treasury inflation-protected securities |
| `fixed income` | `bill` | Bill |
| `fixed income` | `note` | Note |
| `fixed income` | `medium term note` | Medium-term note |
| `fixed income` | `float rating note` | Floating-rate note |
| `fixed income` | `asset backed security` | Asset-backed security |
| `fixed income` | `bond with warrants` | Bond with warrants |
| `fixed income` | `depositary receipt on debt` | Depositary receipt on debt |
| `fixed income` | `preferred convertible` | Preferred convertible |
| `loan` | `loan` | Loan receivable |
| `mutual fund` | `mutual fund` | Mutual fund |
| `mutual fund` | `fund of funds` | Fund of funds |
| `other` | `other` | Other investment type |
| `other` | `hedge fund` | Hedge fund |
| `other` | `private equity fund` | Private equity fund |
| `other` | `null` | Unknown or missing classification |

Source: [Plaid Investments API](https://plaid.com/docs/api/products/investments/)

**Normalization rule:**

1. If `type` is already one of the 9 values in [enum_investment_security_type](#enum_investment_security_type), use it as-is.
2. Else if `subtype` is present, map via [enum_investment_security_subtype](#enum_investment_security_subtype).
3. Else → `other`.

Each holding is assigned 100% of `institution_value` to one type bucket. Instrument type reflects Plaid classification, not underlying fund composition (e.g. a bond ETF with `type = etf` counts as `etf`).

---

### enum_loan_status_type

`plaid_liabilities_student.loan_status_type` — student loan status.

| Value | Description |
| --- | --- |
| `cancelled` | Loan cancelled |
| `charged off` | Loan charged off |
| `claim` | Claim status |
| `consolidated` | Loan consolidated |
| `deferment` | Payment deferred |
| `delinquent` | Payment delinquent |
| `discharged` | Loan discharged |
| `extension` | Payment extension |
| `forbearance` | Forbearance |
| `in grace` | In grace period |
| `in military` | Military deferment |
| `in school` | In-school deferment |
| `not fully disbursed` | Not fully disbursed |
| `other` | Other status |
| `paid in full` | Paid in full |
| `refunded` | Refunded |
| `repayment` | In repayment |
| `transferred` | Loan transferred |
| `pending idr` | Pending income-driven repayment |

Source: [Plaid Liabilities API](https://plaid.com/docs/api/products/liabilities/)

---

### enum_interest_rate_type

`plaid_liabilities_mortgage.interest_rate_type` — mortgage interest rate type.

| Value | Description |
| --- | --- |
| `fixed` | Fixed interest rate |
| `variable` | Variable interest rate |

---

## API Field Reference

Plaid response field documentation. Datatable column names map to these fields via underscore flattening (see [Data Tables](#data-tables)). Account endpoints reference the shared Account object defined below.

---

## Shared Account object

Returned in `accounts[]` by most endpoints. Field availability varies by endpoint — see per-endpoint notes.

### Core fields

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Unique account identifier. Case-sensitive. Changes if account is reconciled differently or Item is re-linked. Closed accounts are not returned. |
| `name` | string | Account name (user-assigned or institution-assigned) |
| `official_name` | string \| null | Official name from the institution |
| `mask` | string \| null | Last 2–4 alphanumeric characters of account number. May be non-unique within an Item. |
| `type` | string | See [enum_account_type](#enum_account_type) |
| `subtype` | string \| null | See [enum_account_subtype](#enum_account_subtype) |
| `holder_category` | string \| null | `personal`, `business`, `unrecognized`. Beta field. |

### `balances` object

| Field | Type | Notes |
|---|---|---|
| `balances.available` | number \| null | Funds available to withdraw. For credit: typically `limit - current`. For depository: `current` minus pending outflows plus pending inflows. For investment: total cash available to withdraw. |
| `balances.current` | number \| null | Total funds in or owed. Credit/loan: positive = amount owed. Investment: total asset value. |
| `balances.limit` | number \| null | Credit limit (credit) or overdraft limit (depository, common in Europe) |
| `balances.iso_currency_code` | string \| null | ISO-4217 code. Null if `unofficial_currency_code` is set. |
| `balances.unofficial_currency_code` | string \| null | For crypto or non-ISO currencies. Null if `iso_currency_code` is set. |
| `balances.last_updated_datetime` | string \| null | ISO 8601 timestamp. Only returned for Capital One (`ins_128026`). |
| `balances.margin_loan_amount` | number \| null | Borrowed funds (margin). Investment endpoints only. |

**Balance freshness:** Cached unless returned by `/accounts/balance/get` or `/signal/evaluate` (Balance-only ruleset). If `current` is null, `available` is guaranteed non-null (and vice versa for `/accounts/balance/get`).

**Insight usage:** `balances.current` for net worth; `balances.available` for spending power. Always null-check `available`.

### Auth-only fields

Present on `/auth/get` accounts. Not mapped to datatables.

| Field | Type | Notes |
|---|---|---|
| `verification_status` | string \| null | Micro-deposit or database verification status (e.g. `automatically_verified`, `pending_manual_verification`) |
| `verification_name` | string \| null | Account holder name used for verification |
| `verification_insights` | object \| null | Database Auth insights (name match score, account number format, ACH return history) |
| `persistent_account_id` | string \| null | Stable ID across Items at TAN institutions (Chase, PNC, US Bank) |

### Identity-only fields

Present on `/identity/get` accounts. Not mapped to datatables.

| Field | Type | Notes |
|---|---|---|
| `owners[]` | array | Account owner info. Each owner has `names[]`, `emails[]`, `phone_numbers[]`, `addresses[]`. Only `names` is guaranteed. |

### Shared response fields

Most account endpoints also return:

| Field | Type | Notes |
|---|---|---|
| `item` | object | Item metadata: `item_id`, `institution_id`, `institution_name`, `webhook`, `error`, product lists |
| `request_id` | string | Unique request identifier for troubleshooting |

---

## `/accounts/get`

Retrieve a list of accounts associated with any linked Item. Returns active bank accounts only (not closed).

**Product:** Accounts (free)
**Account scope:** All active Item accounts
**Balance freshness:** Cached — reflects last successful Item update (~daily if Transactions/Investments/Liabilities enabled)
**Maps to:** `plaid_accounts` (alternate source)

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |

### Response fields — `accounts[]`

See [Shared Account object](#shared-account-object). Does not include `balances.margin_loan_amount`, Auth fields, or `owners[]`.

### Response fields — other

| Field | Type | Notes |
|---|---|---|
| `item` | object | Item metadata |
| `request_id` | string | Request identifier |

### Notes

- Free to use; preferred for account list when real-time balances are not required
- For real-time balances, use `/accounts/balance/get` instead
- Listen for `NEW_ACCOUNTS_AVAILABLE` webhook and use Link update mode for accounts created after initial link

---

## `/accounts/balance/get`

Returns the real-time balance for each of an Item's accounts. Forces `available` and `current` to refresh from the institution.

**Product:** Balance (paid, per-call)
**Account scope:** All active Item accounts
**Balance freshness:** Real-time (sync call; typically <10s, occasionally up to 30s+)
**Maps to:** `plaid_accounts` (primary source)

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |
| `options.min_last_updated_datetime` | string | no | ISO 8601 — oldest acceptable balance timestamp |

### Response fields — `accounts[]`

See [Shared Account object](#shared-account-object). Does not include `balances.margin_loan_amount`, Auth fields, or `owners[]`.

### Response fields — other

| Field | Type | Notes |
|---|---|---|
| `item` | object | Item metadata |
| `request_id` | string | Request identifier |

### Notes

- Recommended for PFM use cases requiring current balances
- `balance` is not a Link product — use any other product to initialize Link, then call this endpoint
- For ACH return-risk assessment, Plaid recommends `/signal/evaluate` instead (out of scope here)
- Use `available` for spending power; use `current` for net worth calculations

---

## `/auth/get`

Retrieve bank account and routing numbers for an Item's checking, savings, and cash management accounts.

**Product:** Auth
**Account scope:** Depository accounts (checking, savings, cash management)
**Balance freshness:** Cached when present
**Maps to:** *(secondary — no datatable)*

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |

### Response fields — `accounts[]`

[Shared Account object](#shared-account-object) plus Auth-only fields (`verification_status`, `verification_name`, `verification_insights`, `persistent_account_id`).

### Response fields — `numbers`

| Field | Type | Notes |
|---|---|---|
| `numbers.ach[]` | array | US ACH: `account`, `account_id`, `routing`, `wire_routing` |
| `numbers.eft[]` | array | Canada EFT: `account`, `account_id`, `institution`, `branch` |
| `numbers.international[]` | array | IBAN/BIC: `account_id`, `bic`, `iban` |
| `numbers.bacs[]` | array | UK BACS: `account`, `account_id`, `sort_code` |

### Notes

- Not used by current insight examples
- `DEFAULT_UPDATE` webhook fires when Auth data changes — re-fetch affected accounts

---

## `/identity/get`

Retrieve account holder information on file with the financial institution.

**Product:** Identity
**Account scope:** All accounts (filterable by `account_ids`)
**Balance freshness:** N/A
**Maps to:** *(secondary — no datatable)*

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |

### Response fields — `accounts[]`

[Shared Account object](#shared-account-object) plus `owners[]` (Identity-only). Each owner contains:

| Field | Type | Notes |
|---|---|---|
| `owners[].names[]` | string[] | Owner names. Always returned. |
| `owners[].emails[]` | object[] | `{ data, primary, type }` |
| `owners[].phone_numbers[]` | object[] | `{ data, primary, type }` |
| `owners[].addresses[]` | object[] | `{ data, primary }` with street, city, region, postal_code, country |

### Notes

- Not used by current insight examples
- Only `names` is guaranteed; other owner fields depend on institution

---

## `/transactions/sync`

Returns incremental transaction updates using a cursor. Preferred over `/transactions/get`.

**Product:** Transactions
**Account scope:** `accounts[]` subset — only accounts with transactions in the response; investment accounts omitted
**Balance freshness:** Cached
**Maps to:** `plaid_transactions`

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `cursor` | string | no | Omit on first call; pass `next_cursor` on subsequent calls |
| `count` | integer | no | Max transactions per page (default 100) |
| `options.account_id` | string | no | Filter to one account (creates separate cursor per account) |
| `options.include_original_description` | boolean | no | Include raw description |
| `options.days_requested` | integer | no | Days of history (up to 730) |

### Response fields — `accounts[]`

Partial [Shared Account object](#shared-account-object). **Do not use as canonical account list.**

### Response fields — `added[]` / `modified[]`

| Field | Type | Notes |
|---|---|---|
| `transaction_id` | string | Unique transaction identifier |
| `account_id` | string | Account this transaction belongs to |
| `amount` | number | Positive = money out (expense/debit). Negative = money in (credit/refund). |
| `date` | string | Posted date (YYYY-MM-DD) |
| `authorized_date` | string \| null | Date transaction was authorized |
| `name` | string | Raw transaction name from institution |
| `merchant_name` | string \| null | Cleaned merchant name (preferred for display) |
| `pending` | boolean | True if not yet posted. Exclude from most analysis. |
| `payment_channel` | string | See [enum_payment_channel](#enum_payment_channel) |
| `personal_finance_category.primary` | string | See [enum_pfc_primary](#enum_pfc_primary) |
| `personal_finance_category.detailed` | string | See [enum_pfc_detailed](#enum_pfc_detailed) |
| `personal_finance_category.confidence_level` | string | See [enum_pfc_confidence_level](#enum_pfc_confidence_level) |
| `location.city` | string \| null | City where transaction occurred |
| `location.region` | string \| null | State/region |
| `iso_currency_code` | string \| null | ISO-4217 currency code |
| `unofficial_currency_code` | string \| null | For non-ISO currencies |

### Response fields — `removed[]`

| Field | Type | Notes |
|---|---|---|
| `transaction_id` | string | ID of transaction to remove from local store |

### Response fields — pagination

| Field | Type | Notes |
|---|---|---|
| `next_cursor` | string | Pass in next request |
| `has_more` | boolean | Keep paginating until `false` |

### Notes

- Supports `credit`, `depository`, and some `loan` accounts (subtype `student` only)
- Investment transactions are not imported — no datatable in the current schema
- Always paginate until `has_more = false`
- `personal_finance_category` is the v2 taxonomy; replaces legacy `category` array

---

## `/transactions/get`

Retrieve user-authorized transaction data. Legacy — prefer `/transactions/sync`.

**Product:** Transactions
**Account scope:** Same partial `accounts[]` subset as sync
**Balance freshness:** Cached
**Maps to:** *(secondary — prefer sync for `plaid_transactions`)*

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `start_date` | string | yes | YYYY-MM-DD |
| `end_date` | string | yes | YYYY-MM-DD |
| `options.account_ids` | string[] | no | Filter to specific accounts |
| `options.count` | integer | no | Max per page (default 100, max 500) |
| `options.offset` | integer | no | Pagination offset |

### Response fields

Same transaction and `accounts[]` fields as `/transactions/sync`. Also returns `total_transactions` for pagination.

### Notes

- All new implementations should use `/transactions/sync`
- Transaction fields map to `plaid_transactions` identically

---

## `/investments/holdings/get`

Returns current investment positions for investment-type accounts.

**Product:** Investments
**Account scope:** Investment-type accounts only
**Balance freshness:** Cached; `accounts[]` includes `balances.margin_loan_amount`
**Maps to:** `plaid_investment_holdings`, `plaid_investment_securities`; investment `accounts[]` enriches `plaid_accounts.balances_margin_loan_amount`

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |

### Response fields — `accounts[]`

[Shared Account object](#shared-account-object) including `balances.margin_loan_amount`. Investment accounts only.

### Response fields — `holdings[]`

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Account holding this position |
| `security_id` | string | Links to `securities[]` |
| `quantity` | number | Shares/units held |
| `cost_basis` | number \| null | Average cost per share |
| `institution_value` | number \| null | Current market value from institution |
| `institution_price` | number \| null | Price per share from institution |
| `institution_price_as_of` | string \| null | Date of price (YYYY-MM-DD) |
| `iso_currency_code` | string \| null | Currency |
| `unofficial_currency_code` | string \| null | For non-ISO currencies |

### Response fields — `securities[]`

| Field | Type | Notes |
|---|---|---|
| `security_id` | string | Join to `holdings[].security_id` |
| `name` | string \| null | Full security name |
| `ticker_symbol` | string \| null | Exchange ticker |
| `type` | string | Summary security type — see [enum_investment_security_type](#enum_investment_security_type); normalize via [enum_investment_security_subtype](#enum_investment_security_subtype) when raw value is not a summary type |
| `subtype` | string \| null | Granular security classification — see [enum_investment_security_subtype](#enum_investment_security_subtype) |
| `isin` | string \| null | ISIN |
| `close_price` | number \| null | Latest closing price |
| `close_price_as_of` | string \| null | Date of close price |
| `iso_currency_code` | string \| null | Currency |

### Notes

- Join `holdings` to `securities` on `security_id`
- `cost_basis` is often null — don't rely on it without a fallback
- `institution_value` is the most reliable current value field

---

## `/liabilities/get`

Returns detailed liability data for credit and loan accounts.

**Product:** Liabilities
**Account scope:** All Item accounts in `accounts[]`; liability detail for credit cards, student loans, mortgages
**Balance freshness:** Cached (~daily)
**Maps to:** `plaid_liabilities_credit`, `plaid_liabilities_student`, `plaid_liabilities_mortgage`

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `access_token` | string | yes | Item access token |
| `options.account_ids` | string[] | no | Filter to specific accounts |

### Response fields — `accounts[]`

Full [Shared Account object](#shared-account-object) for all Item accounts. Join to liability tables on `account_id`.

### Response fields — `liabilities.credit[]`

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `aprs[]` | array | Each: `apr_percentage`, `apr_type`, `balance_subject_to_apr`, `interest_charge_amount` |
| `is_overdue` | boolean \| null | Whether payment is overdue |
| `last_payment_amount` | number \| null | Amount of last payment |
| `last_payment_date` | string \| null | Date of last payment (YYYY-MM-DD) |
| `last_statement_issue_date` | string \| null | Date of last statement |
| `last_statement_balance` | number \| null | Balance on last statement |
| `minimum_payment_amount` | number \| null | Minimum payment due |
| `next_payment_due_date` | string \| null | Date next payment is due |

### Response fields — `liabilities.student[]`

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `disbursement_dates[]` | string[] \| null | Dates loan was disbursed |
| `expected_payoff_date` | string \| null | Projected payoff date |
| `guarantor` | string \| null | Loan guarantor name |
| `interest_rate_percentage` | number | Annual interest rate |
| `is_overdue` | boolean \| null | Whether payment is overdue |
| `last_payment_amount` | number \| null | Amount of last payment |
| `last_payment_date` | string \| null | Date of last payment |
| `last_statement_issue_date` | string \| null | Date of last statement |
| `loan_name` | string \| null | Name/type of loan |
| `loan_status.type` | string | See [enum_loan_status_type](#enum_loan_status_type) |
| `minimum_payment_amount` | number \| null | Minimum monthly payment |
| `next_payment_due_date` | string \| null | Next due date |
| `origination_date` | string \| null | When loan originated |
| `origination_principal_amount` | number \| null | Original loan amount |
| `outstanding_interest_amount` | number \| null | Accrued interest not yet capitalized |
| `payment_reference_number` | string \| null | Payment reference |
| `pslf_status.estimated_eligibility_date` | string \| null | PSLF estimated eligibility |
| `repayment_plan.type` | string \| null | e.g. `income_driven`, `standard`, `graduated` |
| `sequence_number` | string \| null | Loan sequence number |

### Response fields — `liabilities.mortgage[]`

| Field | Type | Notes |
|---|---|---|
| `account_id` | string | Links to `accounts[]` |
| `account_number` | string \| null | Loan account number |
| `current_late_fee` | number \| null | Current late fee owed |
| `escrow_balance` | number \| null | Escrow account balance |
| `has_pmi` | boolean \| null | Whether PMI is present |
| `has_prepayment_penalty` | boolean \| null | Whether prepayment penalty applies |
| `interest_rate.percentage` | number \| null | Interest rate |
| `interest_rate.type` | string \| null | See [enum_interest_rate_type](#enum_interest_rate_type) |
| `last_payment_amount` | number \| null | Last payment amount |
| `last_payment_date` | string \| null | Last payment date |
| `loan_term` | string \| null | Term e.g. "30 year" |
| `loan_type_description` | string \| null | e.g. "Conventional" |
| `maturity_date` | string \| null | Loan maturity date |
| `minimum_monthly_payment` | number \| null | Minimum monthly payment |
| `next_monthly_payment` | number \| null | Next scheduled payment amount |
| `next_payment_due_date` | string \| null | Next payment due date |
| `origination_date` | string \| null | Origination date |
| `origination_principal_amount` | number \| null | Original loan amount |
| `past_due_amount` | number \| null | Amount past due |
| `property_address` | object \| null | Street, city, region, postal code of property |
| `ytd_interest_paid` | number \| null | Interest paid year-to-date |
| `ytd_principal_paid` | number \| null | Principal paid year-to-date |

### Notes

- Supported types: `credit` (subtype `credit card`, `paypal`), `loan` (subtype `student`, `mortgage`)
- Join liability tables to `plaid_accounts` on `account_id`
- US-only coverage; limited Canadian support
- Credit data refreshed daily; mortgage/student loan data may lag
