# Reference workflow: Invoice → HubSpot Deal Sync

A sort-of-realistic read-from-A, write-to-B workflow for discussing and testing the
format in `../../n8n-workflow-testing-prd.md`. An external experiment runs this
reference on n8n 2.41.7. See the [execution report](execution-report-n8n-2.41.7.md).

| File | What it is |
|---|---|
| `invoice-deal-sync.workflow.json` | The workflow, importable into n8n. No credentials attached. |
| `invoice-deal-sync.n8n-test.yaml` | Six test cases in the draft format |
| `mocks/` | Response bodies loaded with `bodyFileName` |
| `execution-report-n8n-2.41.7.md` | Recorded results, execution scope, and a fixed-version run pack |

## The workflow

```
Webhook → Fetch Invoices ─┬→ Collect Invoices → Any Invoices? ─┬→ Split Invoices → Filter Eligible ─┬→ Match Company → Build Deal Update → Update Deal → Audit Log
                          │                                    └→ No Invoices                     └→ Lookup Company ┘
                          └(error)→ Alert Billing Channel
```

| Node | Type | What it exercises |
|---|---|---|
| Invoice Sync Webhook | Webhook | trigger input (`since`, `region`) |
| Fetch Invoices | HTTP Request, paginated, retries 3× | response sequences; a retry uses up the next response in the sequence; error output |
| Alert Billing Channel | Slack | a request that must happen only on failure |
| Collect Invoices | Code | flattens pages into one count |
| Any Invoices? / No Invoices | IF / No-op | which branch ran; early exit |
| Filter Eligible | Filter (region, currency, has deal ID) | checking *which* items survive; one decoy invoice per condition |
| Lookup Company | **HubSpot** company search by domain | a built-in service node mocked at the HTTP layer; `when:` needed because one node makes calls to different URLs |
| Match Company | Merge by domain | an invoice with no matching company is dropped |
| Build Deal Update | Set, with field references | where a misspelled field reference would live |
| Update Deal | **HubSpot** deal update | field-level assertions on the outbound request |
| Audit Log | Postgres insert | a node the HTTP hook can't see; mocked by output only |

**To demonstrate a silent failure:** in Build Deal Update, change `$json.amount_due` to
`$json.amountDue`. The workflow still succeeds, and Update Deal sends `amount: "NaN"` to
HubSpot. Cases 1 and 2 fail on the request body. Change `$json.hubspot_deal_id` to
`$json.hubspotDealId` and the request URL becomes `/deals/v1/deal/` with no ID, which the
URL assertions catch.

## Format choices this file makes (provisional)

Matching by node follows PRD §7.6. The rest go beyond PRD §7 and need a decision before
they become the format:

- **`defaults.mocks`** shared across cases. Without it, every case repeats the Postgres
  output mock, because pre-flight (§6.3) requires a mock for every node the HTTP hook
  can't see, in every case.
- **Mapping vs. list for `respond`:** a single mapping is reused for every call; a list is
  a sequence that fails when it runs out.
- **Mocks match by node, plus an optional `when:`.** `when:` is needed only where one node
  calls different URLs (Lookup Company).
- **`bodyFileName`**, borrowed from WireMock, path relative to the spec file.
- **Request assertions split `url` from `query`.** Query parameter order isn't reliable,
  so `query` is a subset match on its own.
- **`pluck`** to compare one field across all items or calls, so the check covers which
  items survived, not just how many.
- **`assert`** as an n8n expression for anything the structured checks can't express. It
  introduces a new expression variable, `$request`. HubSpot's v1 deal API sends properties
  as a `[{name, value}]` array; selecting the amount from it needs the expression form.

## Findings from building it

The findings below came from reading the n8n source at `master` `0e1c754999`
(2.43.0 in development). The [execution report](execution-report-n8n-2.41.7.md)
records the separate runtime checks on n8n 2.41.7.

- **Mocked error statuses already behave like real ones.** `callEvalMockHandler` throws an
  error shaped like an Axios or legacy request-library error for status ≥ 400, unless the
  request sets `ignoreHttpStatusErrors`
  (`packages/core/src/execution-engine/eval-mock-helpers.ts:207-266`). Retries,
  `continueErrorOutput` and `NodeApiError` handling should work unchanged.
- **Pagination passes through the hook.** `requestWithAuthenticationPaginated` calls
  `helpers.requestWithAuthentication` / `helpers.request` for each page (`pagination.ts:135,142`),
  and both are hooked.
- **HubSpot App Token** goes through `requestWithAuthentication` with a full
  `https://api.hubapi.com` URL (`Hubspot/V2/GenericFunctions.ts:34,43`). Search-by-domain is
  `POST /companies/v2/domains/{domain}/companies`; deal update is
  `PUT /deals/v1/deal/{id}`, which sends `dealstage` before `amount`.
- **No network-error responses.** `EvalMockHttpResponse` is `{ body, headers, statusCode }`
  only. "Endpoint unreachable" and timeouts can't be mocked without extending it (PRD §7.5).

## Runtime questions

The [execution report](execution-report-n8n-2.41.7.md#answers-to-the-readmes-runtime-questions)
answers the first four questions on n8n 2.41.7. Retry timing remains a separate measurement.

- Whether the Slack node sends `channel` as `#billing-alerts` or resolves it to an ID first.
- What the error item looks like on Fetch Invoices' error output (the Slack text uses
  `$json.error?.message` defensively).
- Whether Merge v3 accepts a dotted path (`properties.domain.value`) in `field2`.
- The Filter's `notEmpty` on a missing `hubspot_deal_id` with loose type validation.
- Retry wait: three tries at 1 s each adds about 2 s to the two retry cases.
