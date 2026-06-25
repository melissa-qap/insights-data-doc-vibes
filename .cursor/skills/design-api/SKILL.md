---
name: design-api
description: >-
  Designs sync and read API surfaces for Plaid-backed financial features.
  Use when the user asks for API design, endpoints, routes, request/response
  contracts, or how to expose insight logic as HTTP APIs.
---

# Design API Surface

Produce API specs for a domain: sync endpoints (Plaid → datatables), read endpoints (datatables → JSON), and client composition for composite screens.

## Relationship to outline-plaid-insight

| Skill | Owns |
|---|---|
| [outline-plaid-insight](../outline-plaid-insight/SKILL.md) | Insight logic — datatables, calculations, output fields, core partials |
| **design-api** (this skill) | Routes, sync vs read layers, request/response contracts, client composition |

Always read the relevant insight spec or core partial before drafting APIs. When building end-to-end: **insight calculation first**, then API surface.

## Resources

| File | Purpose |
|---|---|
| [plaid-api-schema.md](../outline-plaid-insight/plaid-api-schema.md) | Plaid endpoints, datatable mappings, sync sources |
| [outline-plaid-insight examples](../outline-plaid-insight/examples/README.md) | Insight specs and shared core partials |
| [examples/](examples/README.md) | Finalized API specs — add every new domain here |

## Two API layers

| Layer | Calls Plaid? | Writes datatables? |
|---|---|---|
| **Sync APIs** | Yes | Yes |
| **Read APIs** | No | No |

Read APIs query datatables only — never call Plaid at runtime. Sync APIs document which Plaid endpoint and datatable they populate.

Apply [output formatting](../outline-plaid-insight/SKILL.md#output-formatting) from the insight skill to all monetary and fraction-percent response fields.

## Workflow

1. **Clarify scope** — Which screen or domain? Which distinct client queries?
2. **Read insight logic** — Load relevant core partial or insight spec from outline-plaid-insight.
3. **List sync endpoints** — What refreshes which tables? Optional `item_id` scope.
4. **List read endpoints** — One per distinct client query; prefer generic parameterized routes over thin wrappers.
5. **Link calculations** — Each read endpoint references insight core or inline steps.
6. **Document client composition** — Composite screens call multiple read APIs; not server BFFs.
7. **Save** — `examples/{domain}/{domain}-apis.md`; index in [examples/README.md](examples/README.md).

## Output template

Save as `examples/{domain}/{domain}-apis.md`:

```markdown
# [Domain] APIs

### Description
[Domain purpose + screens these APIs power]

### Shared calculation
[Link to insight core partial if applicable]

### Sync APIs

#### [METHOD] [route]
- **Plaid source:** ...
- **Datatable writes:** ...
- **Request / response:** ...
- **Powers:** ...

### Read APIs

#### [METHOD] [route]
- **Parameters:** ...
- **Calculation:** [link to core partial or steps]
- **Response:** [field table]
- **Powers:** [UI element]

### Client composition
[Which endpoints a screen calls; invariants between responses]
```

Keep specs concise; move deep SQL to insight core partials.

## Calibration examples

| Domain | File | Endpoints |
|---|---|---|
| Net worth | [net-worth-apis.md](examples/net-worth/net-worth-apis.md) | `POST /v1/plaid/sync/balances`, `GET /v1/accounts`, `GET /v1/performance-history`, `GET /v1/account-balance`, `GET /v1/assets-liabilities` |
| Investment account | [investment-account-apis.md](examples/investment-account/investment-account-apis.md) | `POST /v1/plaid/sync/balances` (cross-ref), `POST /v1/plaid/sync/investment-holdings` |
| Cash flow | [cash-flow-apis.md](examples/cash-flow/cash-flow-apis.md) | `POST /v1/plaid/sync/transactions`, `GET /v1/cash-flow/recurrences` |
