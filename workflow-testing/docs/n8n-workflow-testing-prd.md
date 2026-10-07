# PRD: End-to-End Workflow Testing for n8n

**Status:** Draft / internal working document
**Date:** August 2026 (review findings folded in October 2026)
**Target n8n version at time of research:** 2.34.0 (`9d9e9bf97e`)
**Author:** James Shannon

**Forum discussion:** [Core FR & Proposal: Workflow Test Harness & Format](https://community.n8n.io/t/core-fr-proposal-workflow-test-harness-format/306721/3)

> This is the internal design record — findings, rationale, rejected approaches, and open
> questions.

---

## 1. Purpose

Design an end-to-end testing framework for n8n workflows, intended as an **upstream
contribution** to n8n. It must work against a live, hosted
(self-hosted or cloud) n8n instance.

The document records what already exists in the n8n codebase, what was tried before and
removed, the architecture that follows from those constraints, and the decisions we've
settled or deferred.

---

## 2. Problem Statement

### 2.1 What breaks

n8n workflows fail silently in a way that most software does not. Change one field
reference in one node and nothing tells you. The workflow keeps running, the execution is
marked `success`, and it quietly writes wrong data downstream — potentially for weeks
before anyone notices.

This happens through both authoring paths: editing in the UI, and editing workflow JSON
directly.

### 2.2 Why n8n specifically

In a typed language this class of bug barely exists. `node2.output.customer_id` with a
typo is a compile error.

In n8n, `{{ $('Node 2').item.json.customer_id }}` is a **string evaluated at runtime**.
There is no build step and no static analysis. A misspelled field resolves to `undefined`
and propagates downstream without raising.

A system built on runtime-evaluated string references with no type checker has **no
mechanism at all** for catching wiring errors. Tests are the only place that check can live.

The editor's expression preview can surface an unresolved reference while someone is
looking at it with execution data loaded (not verified in code). Nothing catches it **at
save time or in CI**.

### 2.3 Two modes, one mechanism

Testing here serves two related goals:

| Mode | Goal | Cost to author | Failure meaning |
|---|---|---|---|
| **Characterization** (snapshot) | Detect that behavior *changed* | Near zero — generated from a run | Ambiguous: regression or intended change |
| **Specification** | Assert behavior is *correct* | High — a human defines correct | Unambiguous |

These are **the same feature with different assertion density**. Both
need identical infrastructure: supply input, mock external calls, run the workflow,
inspect what happened. Only the assertions differ.

Characterization is the adoption on-ramp: authoring cost is near zero and it catches the
majority of real breakages. Specification is opt-in refinement layered on the same fixtures.

> "The workflow runs without throwing" is
> *not* sufficient for characterization. A misspelled field reference produces `undefined`
> and completes successfully. Characterization still needs assertions — just cheap, generic,
> auto-derivable ones (see §8.2).

### 2.4 Target user

Compliance-driven and enterprise teams. Relevant consequences:

- They need **repeatable, exportable evidence** tied to a workflow version (ISO 27001-style
  change-management controls).
- They will **hand-author on the order of two dozen scenarios per workflow**. The "record a
  run" path is an accelerator, not the dominant authoring mode. This drives the file-format
  decision in §7.

---

## 3. Prior Art: What n8n Built and Removed

n8n built approximately this
feature, ran it for seven months, and removed it.

### 3.1 Timeline

| Date | PR | Event |
|---|---|---|
| Nov 6 2024 | #11505 | `test_definition` table + TypeORM entity |
| Nov 12 2024 | #11591 | Internal API for test definitions |
| Nov 14 2024 | #11691, #11742 | Description field; PK migrated to string nanoid |
| Nov 26–27 2024 | #11831, #11851 | Workflow filter; metrics API |
| **Dec 9 2024** | **#12001** | **`mockedNodes` property added to TestDefinition** |
| **Dec 13 2024** | **#12009** | **Node mocking logic added to the Test Runner** |
| Jan 7 2025 | #12348 | Mocking switched from node names to node IDs |
| Jan 13 2025 | #12541 | Fix node mocking for evaluation executions |
| **Mar 17 2025** | **#13899** | **UI polish on the mock-nodes modal** |
| May 16 2025 | #15194 | Evaluation Trigger + Evaluation nodes land |
| **May 22 2025** | **#15520** | **`CleanEvaluations` migration drops `test_definition`, `test_metric`, `test_run`, `test_case_execution`** |
| May 23 2025 | #15542 | New Evaluations backend |
| May 26 2025 | #15550 | New Evaluations frontend |

Removal scale: **−2,815 lines** backend, **−1,643** frontend. Deleted views included
`TestDefinitionListView`, `TestDefinitionEditView`, `TestDefinitionNewView`,
`TestDefinitionRunDetailView`, `testDefinition.store.ee.ts`, and `NodesPinning.vue` — a
canvas UI for clicking which nodes to mock.

### 3.2 What the feature was

From the deleted entity's own docstring (`test-definition.ee.ts`):

> It combines: the workflow under test — the workflow used to evaluate the results of test
> execution — the filter used to select test cases from previous executions of the workflow
> under test - annotation tag

So the model was:

- **Test corpus** = tagged **past production executions**, selected by annotation tag
- **Assertions** = a **second n8n workflow** you had to build
- **Mocking** = a list of node IDs whose output was pinned from the recorded execution
- **Metrics** = a separate `test_metric` entity

The route was `@RestController('/evaluation/test-definitions')`.

### 3.3 How mocking worked

From the deleted `utils.ee.ts`:

```ts
const nodeData = executionData.resultData.runData[pastNodeName];
if (nodeData?.[0]?.data?.main?.[0]) {
  pinData[nodeName] = nodeData[0]?.data?.main?.[0];
} else {
  throw new TestCaseExecutionError('MOCKED_NODE_DOES_NOT_EXIST');
}
```

Node-**output** pinning, sourced from a recorded execution, `runIndex` 0 and output 0 only.

This also explains the annotation-based pruning exemption found in
`execution.repository.ts` — the corpus *was* annotated executions, so it had to survive
pruning.

### 3.4 Why it was removed

PR **#15520**, authored by Eugene (`burivuhster`), states in its own summary:

> - There's no concept of Test Definition anymore
> - Process of 'setting up evaluation' happens in a workflow itself (by adding specific nodes)
> - **Node mocking feature is out of scope for now, might be re-implemented again in the future**
> - Previously past executions played the role of test cases source. Now test cases would be
>   provided by the Evaluation Trigger. Concept of 'past execution' is obsolete now

Supporting evidence that this was **descoped, not rejected**:

- **It was under active development when removed.** The mock-nodes modal got UI work in March 2025, two
  months before removal. Node-ID migration and mocking bug fixes ran through January.
- **It never shipped.** Gated behind PostHog experiment `025_workflow_evaluation`
  (`// Enable with window.featureFlags.override('025_workflow_evaluation', true)`). Nearly
  every commit is `no-changelog`. Seven months of work, never GA, no public deprecation.
- **The replacement is a different product.** The new Evaluation Trigger landed six days
  *before* the old model was dropped. This was a planned swap toward dataset-driven AI
  evaluation, split out of a branch named `ai-evaluation-mvp-rebased-2` (Linear AI-866).
- **Nobody argued to keep it.** Reviews on #15520: `guillaumejacquart` APPROVED,
  `jeanpaul` APPROVED. No recorded discussion of the mocking removal.

### 3.5 People

| Person | GitHub | Role | Still active |
|---|---|---|---|
| Eugene | `burivuhster` | Wrote the mocking impl (#12001, #12009) **and** the removal PR (#15520) | Yes — commits through Aug 2026 |
| Guillaume Jacquart | `guillaumejacquart` | Approved the removal | Yes — commits through Aug 2026 |
| JP van Oosten | `jeanpaul` | Merged the removal, co-author | No commits since Mar 2025 |
| Dana | `dana-gill` | Authored replacement Evaluation nodes (#15194) | Not in recent history |
| Raúl Gómez Morales | `r00gm` | Mock-nodes modal UI (#13899) | Not checked |

No matching `burivuhster` account was found on `community.n8n.io`, and the forum offers no
mechanism to pull someone into a thread. **Tagging is not available.** Quote #15520 and link
it instead — nearly as effective and cannot misfire.

### 3.6 How our proposal differs

Any pitch should be built on these three differences:

| Old TestDefinition | This proposal |
|---|---|
| Corpus = recorded past executions | Fixtures authored in git |
| Mock = node output pinned at `runIndex` 0 | Mock at the HTTP boundary; node executes for real |
| Assertions = build a second n8n workflow | Assertions declared in the spec |

**The old model could only test scenarios that had already
happened in production.** You cannot record a 500 you never received. The original
community request — simulate empty results, API errors, unreachable endpoints — was
structurally impossible under that design.

---

## 4. Prior Art: The Current Evaluations Feature

### 4.1 What it is

Licensed. `getMaxWorkflowsWithEvaluations()` reads `quota:evaluations:maxWorkflows` and
returns `0` when unset (`license-state.ts:262`); `0` disables. Frontend paywalls on the same
value (`useEvaluationsLicense.ts`).

**Unverified:** what quota a free community license key grants. The code shows only the
unset default. This matters because §4.5 and §14.7 currently disagree (see §14.7).

Existing routes, all `/rest`, all nested under `/workflows/:workflowId/`:

| Route | Methods |
|---|---|
| `/evaluation-configs` | GET list, GET one, POST, PUT, DELETE, `/dataset-candidate`, `/dataset-rows` |
| `/test-runs` | GET list, GET one, GET `/test-cases`, DELETE, POST `/new`, POST `/{id}/cancel` |
| `/eval-collections` | GET, POST, PATCH, DELETE, `/rerun`, `/runs` |
| `/eval-versions` | GET |

Public API mirrors a subset including `POST /workflows/{id}/test-runs` (createTestRun).

### 4.2 How it works

1. Add an **Evaluation Trigger** (`n8n-nodes-base.evaluationTrigger`) pointed at a dataset —
   an n8n **data table** or a **Google Sheet** (the only two sources).
2. Add an **Evaluation** node (`n8n-nodes-base.evaluation`) with one of four operations:
   `setInputs`, `setOutputs`, `setMetrics`, `checkIfEvaluating`.
3. Run from the UI, or `POST /workflows/{id}/test-runs` with optional `concurrency` (1–10)
   and `rowIndices`.

Each dataset row is pinned onto the eval trigger and the workflow runs once per row
(`test-runner.service.ee.ts:249`).

### 4.3 Metric types

| Type | What it does |
|---|---|
| `expression` | **Any n8n expression**, `outputType: 'numeric' \| 'boolean'` |
| `llm_judge` | correctness/helpfulness presets via a chat-model sub-node |
| `string_similarity` | fuzzy text match |
| `categorization` | expected-category match |
| `tools_used` | which agent tools were called |

`expression` + `boolean` is a general pass/fail assertion. **The assertion primitive
partly exists already** — it just requires an AI-shaped scaffold around it.

### 4.4 The `checkIfEvaluating` anti-pattern

A licensed user today prevents a test run from writing to production with the
`checkIfEvaluating` operation (`evaluationUtils.ts:301`):

```ts
if (isEvalTriggerExecuted) return [input, []];   // "we're testing" branch
else                       return [[], input];   // normal branch
```

The sanctioned answer is **to add an if-branch to your production workflow** and build a
parallel no-op path inside it.

Consequences:

- Test-awareness is baked into production logic
- The workflow you test is not the workflow you ship
- Every external call site needs a hand-built bypass
- Nothing asserts what *would* have been sent — the branch just discards it

Frame this as a **boundary, not a defect**. It's reasonable for AI evaluation, where the
model call *is* the thing under test and side-effect suppression is secondary. Those
assumptions invert for read-from-A/write-to-B integration workflows.

This is the strongest single argument for the proposal.

### 4.5 Gaps vs. what we need

- Mocking exists at **one** node (the eval trigger)
- Datasets come from a data table or Google Sheet only — not files versioned in git
- Eval configs are **not exported by source control** (`source-control-export.service.ee.ts`
  exports workflows, variables, data tables, folders, tags, credentials, projects — not eval
  configs). `ExportableWorkflow` also omits `pinData`.
- No assertion on outbound requests at all
- Licensed, so most of the requesting community can't use it

---

## 5. Existing Core Capabilities

**n8n core already contains both mock primitives.** They are used by
internal tooling and never exposed to users. This is exposure work, not new engine work.

### 5.1 HTTP-level interception

`additionalData.evalLlmMockHandler` receives the **fully-built request after credential
auth** and returns a mock response, or `undefined` to fall through to the real network.

Hooked into every path a node can use for HTTP:

| Helper | Hook site |
|---|---|
| `httpRequest` | `request-helpers/factory.ts:69` |
| `request` (legacy request-promise style) | `factory.ts:144`, via `normalizeLegacyRequest` |
| `requestOAuth1` | `factory.ts:195` |
| `requestOAuth2` | `factory.ts:215` |
| `requestWithAuthentication` / `httpRequestWithAuthentication` | `authentication.ts:41,165` |

Traditional REST-wrapper nodes are covered — Telegram uses `this.helpers.request(options)`
(`Telegram/GenericFunctions.ts:242`).

Planted at two sites, both in n8n's internal tooling for testing its own AI builder and
agents (**not** the licensed customer feature):

- `modules/instance-ai/eval/execution.service.ts:556` — `createInterceptingHandler`, for
  workflow evals
- `modules/instance-ai/eval/agent-execution.service.ts:322` — `createRecordingMockHandler`
  (defined at `:538`), for agent tool calls. Alongside it, `EvalMockedCredentialsHelper`
  replaces `additionalData.credentialsHelper` (`:315-321`).

The handler in `core` is not license-gated.

Related existing capability: the handler already accumulates `interceptedRequests: []` per
node (`instance-ai/eval/execution.service.ts:868-872`) — in memory, never persisted.

**Mocked error statuses behave like real ones.** When a handler returns `statusCode >= 400`,
`callEvalMockHandler` (`eval-mock-helpers.ts:201-250`) throws an error shaped like the real
HTTP library's: Axios shape (`isAxiosError`, `response.status/data/headers`) or legacy
request-promise shape (`statusCode`, `response.body`). Nodes can't tell the difference, so
retry-on-fail, `continueErrorOutput`, and `NodeApiError` handling all work unchanged. This
holds as long as the generalized handler keeps routing through `callEvalMockHandler`.

**Paginated requests pass through the hook too.** `requestWithAuthenticationPaginated`
(`request-helpers/pagination.ts:134,141`) calls `helpers.requestWithAuthentication` or
`helpers.request` per page, both of which are hooked. Each page is a separate mockable call.

**No network-error responses.** `EvalMockHttpResponse` (`execution-engine/index.ts:16`) is
`{ body, headers, statusCode }` only. "Endpoint unreachable" and timeouts need the type
extended (see §7.5). That's a core change, small but real.

**A recording mock handler already exists.** Read `createRecordingMockHandler` before
designing record mode (§8). It may be the template for "record a run," or it may show why
recording at this layer is harder than it looks.

### 5.2 Credential synthesis

When a mock handler is present and a node has no credentials configured,
`node-execution-context.ts:332` synthesizes them, and `eval-mock-helpers.ts:27` generates a
throwaway RSA key so JWT-signing nodes (Google service accounts) don't crash before reaching
the interceptor.

**Tests can run against nodes with no production credentials configured.** Claim this in
the proposal.

**Caveat:** this sits next to the credentials system, which CONTRIBUTING keeps in-house
(§15). Using the existing synthesis as-is is fine. Changing it, or wrapping
`credentialsHelper` the way `EvalMockedCredentialsHelper` does, may get routed to the n8n
team.

### 5.3 Node-output mocking (`pinData`)

`workflow-execute.ts:1830` — before running any node, if
`runExecutionData.resultData.pinData[nodeName]` exists, the engine substitutes it and skips
execution. `node-helpers.ts:1265` skips parameter validation for pinned nodes.

Limitations:

- **`runIndex` 0 only** (`workflow-execute.ts:1832`) — a node in a loop gets the same mock
  every iteration
- Honored only in `manual` and `evaluation` execution modes (`workflow-runner.ts:355`).
  `n8n execute --id` uses `cli` mode and ignores pin data entirely.
- Size-capped at 12 MB (`workflow-helpers.ts:39`)

### 5.4 Per-run injection

- `IWorkflowExecutionDataProcess.pinData` — honored at `workflow-runner.ts:356`
  (`data.pinData ?? data.workflowData.pinData`), used per-case by
  `test-runner.service.ee.ts:249`
- `configureAdditionalData` — `execution-engine/index.ts:80`, invoked at
  `workflow-runner.ts:415`

### 5.5 What's missing

1. **The REST API won't accept mocks.** `manual-run.dto.ts` has no `pinData` field, and
   `workflow-execution.service.ts` reads `workflowData.pinData` (the *saved* workflow) in all
   three execution cases (lines 155, 192, 229). The capability exists at the runner; the
   endpoint doesn't expose it. **This is the single blocking gap.**
2. **`configureAdditionalData` is a closure** — it cannot cross the queue-mode boundary to a
   worker. The eval test runner works around this by putting `pinData` (serializable data)
   into `createRunExecutionData({ resultData: { pinData } })`
   (`test-runner.service.ee.ts:273-287`). **Any mock spec must be declarative data, not a
   handler function.**
3. **Mocks don't propagate to sub-workflows.** `workflow-execute-additional-data.ts:560-586`
   copies a specific whitelist to sub-executions (`executeWorkflow`, `rootExecutionMode`,
   `evaluationRunId`, streaming state). The mock handler is not on it — an Execute Workflow
   node's child run would hit the real network. Small, independently correct fix.

   **Precedent:** for agent workflow-tool sub-executions, n8n already propagates the mock
   handler, but through a different seam: `configureToolAdditionalData` on
   `AgentRuntimeInstrumentation` (`modules/agents/agent-runtime-instrumentation.ts:42-50`),
   called once per tool invocation, not through the whitelist. Either model the fix on that
   seam or explain why the whitelist is the right place. A reviewer will ask.
4. **No request recording in execution data** (see §9.1).

### 5.6 Also noted, not on the critical path

- `packages/core/nodes-testing/node-test-harness.ts` — an in-process workflow executor used
  for node tests. Not shipped in the npm tarball (`package.json` `files: ["dist","bin"]`),
  but everything under it (`WorkflowExecute`, `ExecutionLifecycleHooks`, the nodes loaders)
  *is* exported. An out-of-process harness is buildable against published packages.
- `@n8n/backend-network/src/http/fake-outbound-http.ts` — an existing `Route` shape
  (`{ method, pathname, status, body, networkError: 'ECONNREFUSED' }`). Align to it;
  `networkError` is a reminder that "endpoint unreachable" is a case users want.
- `n8n execute-batch` — a stock snapshot-regression CLI with `--snapshot`, `--compare`,
  `--concurrency`, `--githubWorkflow`. No mocking, no per-case inputs, `cli` mode.

---

## 6. Architecture

### 6.1 Mock at the HTTP boundary, not node output

If you mock at the node-output boundary, the mocked node never executes — its URL
expression, auth config, header mapping, pagination, and response parsing are all untested.
For a read-A → transform → write-B workflow where you mock both ends, you are testing only
the transform. **The bug we set out to catch becomes invisible.**

If you mock at the HTTP boundary, the node fully executes and only the socket is faked, and
**the outbound request becomes an assertable artifact**. For a workflow whose
purpose is writing to another system, the outbound request *is* the output. Asserting only
on the response the node returned tests your mock, not your workflow.

Every mature tool in this space (VCR, WireMock, nock, MSW, Pact) treats **request
assertion** as first-class. Make it central.

| Lever | What executes | Use for |
|---|---|---|
| HTTP interception | node's full logic; socket faked | **Default.** Any HTTP-based node |
| `pinData[nodeName]` | nothing — output substituted | Trigger input; non-HTTP nodes; skipping expensive subtrees |
| `NodeTypes` substitution | your fake implementation | Out of scope for v1 |

Secondary benefit: an HTTP response is a shape users already understand from API docs, curl,
or Postman. An n8n item array is proprietary and must be hand-constructed. This is an
adoption argument.

### 6.2 The non-HTTP blind spot

Nodes speaking non-HTTP protocols bypass the interceptor entirely:

| Node family | Driver |
|---|---|
| Postgres | `pg-promise` |
| MySQL | `mysql2` |
| Email (IMAP) | `imap` |
| SMTP, FTP/SFTP/SSH, Kafka/AMQP/MQTT, Execute Command, filesystem | various |

**There is no socket-level backstop.** `SsrfProtectionService` does hostname/IP
allow/block-listing but is consumed *inside* the HTTP client
(`httpRequest(requestOptions, additionalData.ssrfBridge)`), and the DNS resolver is a
per-request service, not a global `dns.lookup` patch. It sits at exactly the same layer as
the mock hook and is blind to the same nodes.

**Cost:** for driver-based nodes you get input control (pin the output) but lose
request assertion entirely. State this plainly rather than letting a reviewer find it.

### 6.3 Pre-flight classification

Interception alone cannot guarantee "nothing escaped." But node types are static and the
whole workflow graph is available, so classify **before** executing:

- **interceptable** — uses HTTP helpers. Mocks apply; strict-fail on unmatched is a real
  guarantee.
- **opaque** — protocol driver, shell, filesystem. Must carry an explicit `output` mock, or
  explicit `allowRealIO: true`, or **the case refuses to start**.
- **inert** — Set, IF, Merge, Filter. No egress; ignore.

```
Cannot run case "empty API result": node "Load Customers"
(n8n-nodes-base.postgres) performs I/O the mock layer cannot
intercept. Provide an `output` mock, or set allowRealIO: true.
```

This converts a silent escape into a loud refusal. Without it, "unmatched requests fail"
would not hold for opaque nodes.

**Classification source**, escalating:

1. **Curated registry** in the framework — node type → egress class. Brittle, misses
   community nodes, ships immediately.
2. **Declared on the node type** — an optional `egress: 'http' | 'driver' | 'none'` on
   `INodeTypeDescription`, set by node authors. Covers community nodes; needs backfilling
   across ~400 nodes.
3. **Unknown means opaque** — anything unclassified is treated as un-interceptable.

**Decision: 1 + 3 for v1, pitch 2 as the direction of travel.** Default-deny on unknown
makes it safe from day one and degrades gracefully.

### 6.4 Unmatched request policy

**Strict-fail by default.** If a request finds no matching mock, the case fails.

A test that silently falls through to the real network can POST to
production. Explicit per-case `passthrough: true` opt-in only, for record mode.

### 6.5 Serializability

Because `configureAdditionalData` is a closure and cannot reach a queue-mode worker, the
mock spec **must be declarative data carried in the execution payload**.

Consequence: **the file format is the wire format is the execution payload.** One
schema, three roles.

### 6.6 CI-level isolation (recommended, not built)

Run the instance under test in a network-isolated container. Catches escapes at a layer n8n
cannot see. `packages/testing/containers` already has the stack. Document as recommended CI
setup; do not propose as a feature.

### 6.7 Rejected: rewriting URLs to a separate mock server

The alternative that needs no core change: export the workflow, rewrite HTTP node URLs to
point at a local mock server, and run the copy. A reply in the forum discussion described a
runner, n8n-check, that works roughly this way ("HTTP Request nodes pointed at a local mock", "runs an
exported slice in an isolated n8n runtime"). We haven't seen its code.

Rejected, for four reasons:

1. **It tests a modified workflow.** The rewritten URL expression is the one you didn't
   ship. The wiring bugs this framework exists to catch (§2.2) can sit in exactly that
   expression. It's a milder form of the `checkIfEvaluating` problem (§4.4).
2. **It mostly reaches only the HTTP Request node.** Service nodes build URLs internally.
   The Salesforce node, for example, sends a relative path (`Salesforce/GenericFunctions.ts:109`)
   resolved against an instance URL cached on the credential. Redirecting it means rewriting
   credentials, not the node.
3. **The test file stops being the whole test.** Responses live in another process. Every
   response should be expressible in the spec file, or in a file it names (§7.5
   `bodyFileName`). A separate server is not that.
4. **Shared state brings non-determinism.** One server serving concurrent cases can
   interleave sequences, with one case consuming another's responses. Mocks carried in each
   run's payload (§6.5) can't collide.

It works today with no change to n8n. If the core team declines the
hook changes, something like it is the fallback, with the limits above stated plainly. Say
that in public rather than dismissing it.

---

## 7. Test Case Format

### 7.1 YAML — and the dependency argument

Trade-off considered:

- **JSON** is native to n8n (workflows, exports, source control) and is legitimately the
  better *stub-mapping* format.
- **YAML** is far better for hand-authoring: comments, readable multiline bodies, low noise.

**The target user hand-authors ~24 scenarios per workflow** (§2.4), so hand-authoring
ergonomics outweigh generated-file consistency. Comments in particular are most of what
makes a test file maintainable a year later, and JSON cannot express them.

**The dependency question resolves in YAML's favor:**

- `yaml` is already a **direct dependency of `packages/cli`** (`packages/cli/package.json:283`),
  plus the root and `@n8n/agents`
- **No JSONC or JSON5 parser exists anywhere in the monorepo** (the only `jsonc` hit is
  `biome.jsonc`, parsed by Biome's own tooling)

So YAML costs zero new dependencies in exactly the package where a spec parser would live;
JSONC would require adding one. The "JSON is native" argument is about *convention*, not
tooling.

**Resolution: YAML is a superset of JSON.** Adopt an existing stub-mapping *schema* while
serializing as YAML — one parser accepts both. The record button emits JSON; humans
hand-author YAML. This also defuses a format-consistency objection: we're accepting a
superset of what n8n already uses, not introducing a competitor.

### 7.2 Borrow, don't invent

**Decision: adopt an existing stub-mapping format's schema** rather than designing one.
Candidates:

| Tool | Format | Matching | Response templating |
|---|---|---|---|
| WireMock | JSON | JSONPath, regex, `equalToJson` w/ ignore-extras | Handlebars over the request |
| Mountebank | JSON | Predicates | `copy` behavior lifts request values into response |
| Pact | JSON | Matchers are the core concept | N/A (contract testing) |
| VCR family | YAML/JSON | Configurable request matchers | Replay only |

WireMock and Mountebank are closest to what we want. Mountebank's `copy` behavior is exactly
"lift a value out of the request into the response."

**Large bodies come from files: `bodyFileName`.** Borrowed from WireMock. Paths are
relative to the spec file. This keeps a 1,000-line API response out of the YAML without
introducing a mock server (§6.7). It applies to mocked responses and to expected request
bodies. Small bodies stay inline, where they're easier to read alongside the case.

### 7.3 Templating: reuse n8n's expression language

Do **not** invent a templating DSL. The moment you support `$input[0].x` you need property
access, array indexing, string manipulation, then conditionals — a large surface to design
and defend. **n8n already has an expression language and users already know it.**

```yaml
respond:
  status: 200
  body:
    type: x
    echoed: "={{ $request.body.customerId }}"
```

Costs almost nothing to implement, zero learning curve, and is a far easier sell than "a
second expression language inside n8n."

### 7.3a Assertions: structured checks first, expressions as the fallback

Status and counts aren't enough. A filter can keep the right *number* of rows but the
wrong ones, so assertions must reach individual fields and item identities.

Same principle as §7.3: **don't invent an assertion syntax.** Something like
`response[1].x[4] | count == 10` looks small but is a new language with its own grammar,
precedence, and error messages. Two layers instead:

1. **Structured checks for the common cases:** `count`, subset match on `body`/`query`,
   `executed`, and `pluck` (collect one path from every item or call, compare the list
   exactly). They read cleanly and produce good failure messages ("expected
   [inv-1001, inv-1002], got [inv-1001]").
2. **An n8n expression for everything else:** `assert: "={{ ... }}"` must evaluate to
   `true`. Under `nodes` it runs once per output item (`$json`); under `requests` once per
   call, with a new `$request` variable for the captured request.

The expression layer already has what's needed. JMESPath ships in n8n
(`packages/workflow/src/jmespath-query.ts`, exposed as `$jmespath()`), and the Evaluations
feature already treats a boolean expression as pass/fail (§4.3). So the example above is
`={{ $jmespath($request.body, 'length(x[4])') === 10 }}`.

An example the structured checks can't express: HubSpot's v1 deal update sends fields as a
`[{name, value}]` array, so "the amount sent" is
`$request.body.properties.find(p => p.name === 'amount').value`.

**Decoys.** A field-level assertion only proves the filter works if the fixture contains
rows the filter must drop. Include one decoy per filter condition, so removing or loosening
any single condition fails the case (credit: the n8n-check reply in the forum discussion).

### 7.4 Example

```yaml
workflow: invoice-sync          # name or id; file lives beside the workflow JSON in git
cases:
  # Covers the early-exit path when the upstream API has nothing for us.
  - name: empty API result takes the early-exit branch
    input:
      json:
        since: "2026-01-01"

    mocks:
      # HTTP-level: the node executes for real, only the socket is faked
      - node: Fetch Invoices
        respond:
          status: 200
          body:
            data: []
            next: null

      # Ordered sequence for pagination / loops.
      # Errors on exhaustion — an unexpected extra call is a bug we want surfaced.
      - node: Fetch Pages
        respond:
          - { status: 200, body: { page: 1 } }
          - { status: 200, body: { page: 2 } }

      # Node-output level: for nodes the HTTP hook cannot see (Postgres here)
      - node: Load Customers
        output:
          - json: { id: 1, name: "Acme" }

    expect:
      status: success
      requests:
        - node: Push to CRM
          count: 1
          body:                       # subset match, NOT deep equality
            customerId: 1
            total: 0
        - node: Send Alert
          count: 0                    # negative assertion
      nodes:
        No Invoices Branch: { executed: true }
        Process Invoices:   { executed: false }
```

### 7.5 Schema notes

- **`name` is mandatory** — JUnit's `<testcase>` requires a name attribute; this is what
  appears in CI output. It is a test name, not a comment, and does not replace real comments
  (which are positional and arbitrary).
- **Subset matching, not deep equality.** Captured requests contain timestamps, UUIDs,
  `$now`. Deep-equality comparison flakes within a week. Assert the fields you named;
  optional `ignore` paths for full-body compare.
- **Response sequences error on exhaustion**, not repeat-last.
- **Retries consume from the sequence**, letting retry behavior be tested deliberately.
- Naming: **do not call the file `workflow-eval.json`** — "eval" is taken by the existing
  feature. Use e.g. `invoice-sync.n8n-test.yaml`.
- **"Endpoint unreachable" needs a representation.** The forum discussion names it as a
  case users want, but the schema above can't express it. Borrow `networkError: 'ECONNREFUSED'` from
  `fake-outbound-http.ts`'s `Route` shape (§5.6) as an alternative to `status`/`body` in
  `respond`. Timeouts likely need the same treatment. Requires extending
  `EvalMockHttpResponse` in core (§5.1).
- **`respond` as mapping vs. list.** A single mapping is reused for every matching call; a
  list is a sequence that fails the case when it runs out.
- **`defaults.mocks`** at file level, overridable per case. Needed because pre-flight
  (§6.3) demands an output mock for every opaque node (e.g. Postgres) in every case. At ~24
  cases per workflow, repeating those is noise.
- **Request assertions split `url` from `query`.** Query-parameter order isn't reliable, so
  `query` is its own subset match.

### 7.6 v1 scope: deliberately dumb

**Mocks match by node. Optional `when:` is an exact match on method + URL. Static response
bodies. No matchers, no templating.**

A mock with only `node` answers every request that node makes. Most HTTP nodes make one
kind of call, and the node name is already in the spec. `when:` narrows a mock for a node
that makes several kinds of request. In the reference workflow (§7.7) it was needed once: a
HubSpot lookup node calling a different URL per item.

Most cases need nothing more, the record button generates exact matches anyway, and
matching/templating expands forever once opened. Mention them as "later," not v1 — it keeps
the proposal small, which makes acceptance more likely.

### 7.7 Reference workflow

`reference/invoice-deal-sync/` holds a sort-of-realistic workflow and six cases written in
the draft format, for discussing the format concretely and for sharing (requested in the forum
discussion).

- **Workflow:** webhook → paginated billing API with retries → filter (region, currency,
  has deal ID) → HubSpot company search → merge → set → HubSpot deal update → Postgres
  audit; Slack alert on the billing API's error output.
- **Why HubSpot rather than Salesforce:** HubSpot's App Token auth is a static Bearer
  token with full `https://api.hubapi.com` URLs through `requestWithAuthentication`.
  Salesforce JWT auth needs a token exchange first (unclear whether the hook sees it) and
  uses relative URLs.
- **Cases:** normal sync; pagination with one decoy per filter condition; no matching
  company; empty result; 500 then retry success; all retries fail → alert, nothing
  written.
- **Silent-failure demo:** misspell `amount_due` in the Set node, and HubSpot is sent
  `amount: "NaN"` while the execution succeeds.

Nothing runs it yet. Its README lists node behavior still to confirm by import.

---

## 8. Recording ("Add this run as a test case")

### 8.1 Record generously, assert conservatively

These are two separate decisions.

**Recorded** — near-total: trigger input, plus every intercepted request/response pair per
node. That's what makes the case replayable. Storage is cheap; you can't add data back later.

**Asserted** — much smaller. Asserting every node's full output produces a test that fails
on the next run regardless of correctness (timestamps, UUIDs, `$now`, cursors, ETags). That
is the classic snapshot death spiral: noisy tests get ignored, then deleted.

### 8.2 Default assertions

Descending signal-to-noise:

1. **Execution path** — which nodes ran, in what order, how many times. Extremely stable,
   tiny, catches branch-logic regressions. Nearly free.
2. **Terminal outbound requests** — what the workflow did *to the world* is its actual
   contract.
3. **Shape, not values, for intermediate nodes** — assert expected keys and types, not data.
   This closes the §2.3 gap: a misspelled reference yields a missing/`undefined` key, which
   a shape assertion catches without failing on every changing value.

### 8.3 Two cheap improvements

- **Record twice, diff, auto-ignore.** Run the workflow twice when generating a case;
  anything that differs between runs is volatile by definition and is excluded from
  assertions automatically. Solves most of the noise problem invisibly.
- **Promote to assertion.** The recording holds everything; the generated test asserts the
  conservative subset; the UI lets you click any captured value and promote it. Tightening is
  one click, loosening is deleting a line. Mirrors jest snapshots + explicit `expect`s, which
  is why that combination survives real teams.

So the button is "add this run as a test case, here's what I'll watch, adjust as you like."

### 8.4 Anti-pattern boundary

Seeding a fixture from a recorded execution as a **starting point you then edit** is fine and
is how all snapshot tooling works.

Sourcing the test corpus **from stored past executions** is the removed model (§3.2). Keep
recording as an authoring convenience, **never** as the storage model, and make the
distinction visible — otherwise anyone who remembers TestDefinition will think we're
re-proposing it.

---

## 9. Results, Storage, and Reporting

### 9.1 Execution traces are insufficient

`ITaskData` (`interfaces.ts:3351`) records per node: `data` (input/output items),
`executionTime`, `executionStatus`, `error`, metadata. **There is no record anywhere of the
HTTP requests a node made.** The most valuable assertion in a write-to-B workflow is
invisible in an execution trace.

### 9.2 Pruning

`executions.config.ts:107-118`: `pruneData: true`, `pruneDataMaxAge: 336` (14 days),
`pruneDataMaxCount: 10_000`.

`ExecutionData.data` is a single `text` column — the whole run is one serialized blob, so
request data cannot be pruned or queried separately from the rest.

### 9.3 Two tiers, not three

- **Evidence → the execution record**, pruned normally. Requires a mandatory size cap.
- **Verdict → a small separate table**, never pruned: pass/fail, per-assertion results,
  `workflowVersionId`, spec snapshot, timestamps. This is the compliance artifact and the
  JUnit source.

An earlier draft proposed a separate evidence table. **Rejected as over-engineering.** Two
weeks is enough to debug a failure; only the verdict must outlive that.

n8n already established this pattern. `TestCaseExecution`'s own docstring:

> Entries in this table are meant to outlive the execution entities, which might be pruned
> over time. This allows us to keep track of the details of test runs' status and metrics
> even after the executions are deleted.

with `@OneToOne('ExecutionEntity', { onDelete: 'SET NULL', nullable: true })` and
denormalized `inputs`, `outputs`, `metrics`, `status`, `errorCode`, timings.

**Storage compatibility is high.** `TestCaseExecution.metrics` is
`Record<string, number | boolean>` — booleans allowed, so pass/fail fits. `inputs`/`outputs`
are plain `JsonObject`. Only per-assertion detail (expected/actual/diff) needs a new field.

**Provenance:** `TestRun.workflowVersionId` and `TestRun.evaluationConfigSnapshot` already
exist — the latter is precedent for snapshotting the spec onto the run, which is better than
hashing it. Without both workflow version and spec version, "case X passed on date Y"
establishes nothing.

**Asymmetric retention:** a passing case does not need its evidence kept — store the verdict
and a hash. A failing case needs the full diff. Most cases pass most of the time, so this is
most of the storage problem solved. Design it in from the start.

### 9.4 The prune-budget collision

`execution.repository.ts:543-553` finds the 10,001st newest execution by id and deletes
everything older, **globally across the instance**.

A suite of 200 cases running on every PR does thousands of executions weekly. **Test runs
would evict production execution history.** Someone debugging a real incident finds their
trace gone because CI ran the suite forty times that day.

A reviewer who finds this first could reject the PR on it. **Raise it ourselves.**

An exemption mechanism already exists — annotated executions are excluded both from the count
ranking (`:542`) and from deletion (`:566`) — but `ExecutionAnnotation` is user-facing
(vote/note/tags), so reusing it would make every test execution appear annotated in the UI.
Wanted: the same mechanism with different semantics.

### 9.5 Reporting

JUnit XML generated from the verdict layer. The verdict layer should be designed as
*exactly* what a report needs.

---

## 10. Security and Privacy

### 10.1 Captured requests always contain the secret

Every captured request carries a live `Authorization` header, API key, or signed token.
This is **inherent**: the hook fires *after* credential application
(`authentication.ts:41,165`), which is why the mock is high-fidelity.

### 10.2 The existing redaction subsystem redacts at the wrong moment

`packages/cli/src/modules/redaction/` provides:

- Per-workflow `redactionPolicy`: `none | manual-only | non-manual | all`
- Instance-level `redactionFloor`: `off | production | all`, with workflows blocked from
  going weaker (the 422 in `redaction-policy.ts:30`)
- Strategies: `full-item-redaction`, `node-defined-field-redaction`
- `sensitiveOutputFields?: string[]` on the node type description — "always redacted
  regardless of workflow policy or user permissions and never revealable"
  (`interfaces.ts:2968`)
- Fail-closed application in lifecycle hooks (`execution-lifecycle-hooks.ts:349-377`)

But `processExecution` is called only on **push and read** paths. The
`updateExistingExecution` save paths do not touch it. **Execution data is stored raw and
redacted when served.** Defensible for ordinary node I/O; not for a working credential in
the DB and every backup.

### 10.3 Decisions

- **Redact at capture time**, not read time. A departure from the existing model, justified
  by what's being stored.
- **Mask by known value, not field-name heuristic.** We hold the actual credential at the
  point of application. `eval-mock-helpers.ts:36` already does the heuristic version
  (`SECRET_NAME_PATTERNS` matching `key|secret|token|password|...`), which is the wrong tool
  when the real secret is available to compare against.
- **Default to metadata only** — method, URL, header *names*, body size and hash. Full bodies
  are explicit per-node opt-in. Most assertions ("we POSTed once to /invoices") don't need
  bodies.
- **Separate nullable `requestCapture` column** on `execution_data`. Gives: independent
  pruning, independent size cap, **column omission on normal reads** (exposure requires a
  deliberate query), and a clean permission gate.
- **Cap and truncate.** Precedent: pinned data capped at 12 MB (`workflow-helpers.ts:39`).

### 10.4 Sequencing consequence

**Scope request capture to test runs first.** Bounded context, and core auto-synthesizes fake
credentials for unconfigured nodes (§5.2). General production capture becomes a separate
later proposal with capture-time redaction as its centerpiece.

An earlier draft proposed making request capture a **general production** feature ("what did
this node actually send?" is a top debugging question). **Reversed.** Storing live
credentials at rest across every production execution on a multi-tenant instance draws a long
security review and may reasonably be declined.

---

## 11. API Design

### 11.1 Constraints from the existing shape

- Test runs are **nested under workflows**, not a top-level resource. `EvaluationConfig` is
  nested identically (`workflowId` column, unique on `(workflowId, name)`). A top-level
  `/tests` resource would be the first thing questioned. **Note: `/workflows/{id}/tests` does
  not exist today** — it was only ever a sketch.
- **Fully async.** The controller does sync setup and returns `202` with a `testRunId`,
  detaching case execution (`test-runs.controller.ee.ts:225-236`). Public API returns `201`
  and guards the detached promise (`evaluations.handler.ts:161-163`). Callers poll.
- **Permissions:** `publicApiScope('testRun:create')` + `projectScope('workflow:execute')`,
  with the comment "starting a run triggers real executions." That reasoning transfers.
- **API-key auth is public-API only.** `ApiKeyAuthStrategy` is registered for the public API
  and MCP (`server.ts:190`, `public-api/index.ts:252`), not `/rest`. `/rest` needs a session
  cookie.

### 11.2 The naming collision

`POST /workflows/:id/test-runs` **already means** "run the dataset through the eval trigger
and score it." We cannot claim that path. Two options:

- **New noun** (`/workflow-test-runs`, `/checks`). Clean separation, no entanglement with a
  licensed feature; costs users two things called "tests."
- **Extend the existing concept** — one test run, two case sources. `TestRun.evaluationConfigId`
  is already nullable. Reuses `TestRun`, `TestCaseExecution`, run-list and run-detail UI, and
  the public API surface. Better design, less duplication. Complication: `assertEvaluationsEnabled()`
  gates the whole path today, so per-run-type gating would be needed.

### 11.3 Recommended cut

**Share the entities and orchestration; separate the API surface.** A distinct POST endpoint
writing `TestRun`/`TestCaseExecution` rows with a type discriminator gets all the expensive
reuse (concurrency, cancellation, persistence, prune behavior) with no union-typed request
body.

If a single endpoint were wanted, in-repo precedent exists: `ManualRunDto` dispatches among
three payload shapes via `schemaFor(data)`, and both `EvaluationMetric` and `datasetRefSchema`
are `z.discriminatedUnion`s.

### 11.4 What actually diverges

Narrower than "two data types":

- **Shared:** run status, timings, `workflowId`, `workflowVersionId`, error columns,
  cancellation fields; case-level `executionId` link, status, `runIndex`, timings,
  `inputs`/`outputs`
- **Diverges:** the POST payload (case source), and per-case result detail (per-assertion
  expected/actual/diff + captured requests)

---

## 12. Phased Delivery

Framed as **release cadence and product vision**, not a PR plan. Each phase ships, proves
itself in production, and de-risks the next. Naming is by user-visible capability.

### v1 — Run your test suite from CI

CI reads the spec file, iterates cases itself, calls the run endpoint once per case with
mocks attached, collects captured requests, writes JUnit.

**n8n never learns the word "test."** Core changes collapse to: *accept a mock spec on a run,
and return captured requests.* No entities, no migrations, no scheduling, no concurrency
management, no cancellation semantics, no UI.

Everything else — file format, case orchestration, assertions, reporting — lives in an npm
package we own and iterate on outside n8n's release cycle.

This is the realistic first contribution from an outside contributor.

Core work: generalize the mock hook (rename `evalLlmMockHandler` → neutral); declarative mock
rule type + resolver carried in execution data; accept mocks + trigger input per run; request
capture with capture-time redaction; sub-execution propagation fix; pre-flight node
classification.

### v2 — n8n runs the suite, results in the UI

n8n holds the case list. Adds: case iteration with concurrency limits, per-case aggregation
into a run, cancellation coordinated across mains, failure isolation, verdict persistence
outliving pruning, the prune-budget answer.

**This is most of the remaining work.** Extending the existing `test-runs` concept rather than inventing a new noun shrinks it
considerably.

### v3 — Author and manage tests in the editor

Spec CRUD, a UI, the "add this run as a test case" button with promote-to-assertion. Requires
source-control export so specs version with workflows (fixing the existing gap where eval
configs don't).

Comparatively cheap once v2 exists.

---

## 13. Anti-Patterns and Explicit Non-Goals

| Anti-pattern | Why | Instead |
|---|---|---|
| Sourcing the corpus from stored past executions | The removed model; can only test what already happened | Fixtures in git; recording as an authoring aid only |
| In-workflow test branches (`checkIfEvaluating` style) | Test-awareness in production logic; the shipped workflow ≠ the tested one | Mock below the node |
| Mocking node output by default | The mocked node never executes; node-config bugs invisible | HTTP boundary; node output as fallback only |
| Deep-equality assertions | Timestamps/UUIDs/`$now` flake within a week | Subset match; shape not values for intermediates |
| Passthrough on unmatched requests | A test can POST to production | Strict fail + pre-flight classification |
| Inventing a matcher or templating DSL | Unbounded surface to design and defend | Borrow WireMock/Mountebank/Pact semantics; reuse n8n's `{{ }}` |
| Inventing an assertion mini-language (`a[1].x \| count == 10`) | Same unbounded surface, for assertions | Structured checks + n8n expression `assert`, with `$jmespath()` available (§7.3a) |
| Rewriting URLs to a separate mock server | Tests a modified workflow; misses service nodes; responses live outside the spec; shared state across cases (§6.7) | Intercept below the node; responses in the spec or `bodyFileName` |
| Asserting only status and count | A filter can keep the right number of the wrong rows | `pluck` identities; decoy rows per filter condition |
| A top-level `/tests` resource | Contradicts how n8n nests test runs and eval configs | Nest under `/workflows/:id/` |
| Storing specs only in the DB | Won't version with git-based workflows; repeats the eval-config gap | Spec in git; snapshot onto the run |
| A separate evidence table | Over-engineering; execution pruning already fits | Evidence in the execution record; verdict separate |
| General production request capture in v1 | Live credentials at rest, multi-tenant, long security review | Scope to test runs; propose prod capture separately |
| Asserting every node's full output | Fails on every run; snapshot death spiral | Path + terminal requests + shape |
| A big-bang PR | `CONTRIBUTING.md:529` caps PRs at ~1000 lines | Sequenced small PRs |
| Naming files `*eval*` | Collides with the existing feature | `*.n8n-test.yaml` |

---

## 14. Open Questions

1. **Does n8n require a UI for this to be acceptable?** The single biggest fork in effort. If
   yes, v3 is mandatory and scope roughly doubles. Entirely their call — ask early and
   directly.
2. **New noun or extend `test-runs`?** (§11.2) Affects how much orchestration is reused and
   whether a free feature entangles with a licensed one.
3. **Prune-budget resolution.** Exempt test executions from the count budget, or make the
   config test-aware?
4. **Authorization.** Accepting mocks means `workflow:execute` can change what a run *does*,
   which it currently cannot. Does that warrant a new scope?
5. **Node egress classification** — will n8n accept an `INodeTypeDescription` field, or must
   we ship a curated registry indefinitely?
6. **Which stub-mapping schema** to borrow (WireMock vs. Mountebank vs. Pact matchers).
7. **Licensing.** *Decision: do not raise it.* Evaluations is usable under the community
   license, so the status quo favors us; asking permission invites a "no" nobody was going to
   volunteer. If they want to block it, they will say so.

   > **Contradiction to resolve:** §4.1 and §4.5 say Evaluations is licensed with a default
   > quota of 0, so "most of the requesting community can't use it." This item says it is
   > usable under the community license. Both can't be true, and this decision rests on the
   > second claim. Check what a free community license key grants (§4.1), then fix whichever
   > section is wrong. The decision may survive either way, but its stated reason may not.
8. **Credentials boundary.** Can v1 avoid touching the credentials helper entirely, relying
   only on the existing synthesis (§5.2) and doing capture-time masking (§10.3) outside the
   credentials system? If not, that piece is likely n8n-team work (§15).
9. **v1 has no API entry point.** The public API cannot run a workflow. Its only POST
   endpoints under `/executions` are retry and stop, and under `/workflows` only lifecycle
   actions (activate, publish, archive, …). The one public "run" is the licensed
   `POST /workflows/{id}/test-runs`. So "CI calls the run endpoint" (§12 v1) means one of:
   a new public-API endpoint that runs a workflow (a product decision n8n has so far not
   made), CI driving `/rest` with a session cookie (an internal, unversioned API), or going
   through the licensed Evaluations path. Each is the core team's call.

---

## 15. Contribution Process Constraints

From `CONTRIBUTING.md`:

- **Line 458:** "Feature PRs that arrive with no prior discussion will be closed with a
  pointer to the forum." A forum topic is mandatory before code. The existing community
  thread is a feature *request*, not agreement to accept an implementation.
- **Line 529:** "Each PR should add no more than 1000 lines and cover one logical change."
- **Line 466:** The n8n team handles the most widely used nodes in-house (HTTP Request, Code,
  Webhook, Form, Schedule). The Evaluation node isn't listed, but modifying a shipped node is
  a harder sell than adding capability beside it. **Propose additively.**
- **Line 462** (and the directory listing at line 53): "Contact n8n before starting any change
  under /packages/core." Line 457 names the forum topic as the way to make contact; the
  forum discussion is that contact.
- **Line 464:** "The n8n team handles changes to identity and access management and to the
  credentials system … we keep these in-house." This design touches credentials in three
  places: credential synthesis (§5.2), masking captured requests by real credential value
  (§10.3), and potentially wrapping `credentialsHelper` as `EvalMockedCredentialsHelper`
  does (§5.1). This is a plausible reason for part of the work to be kept in-house. See §14.8.
- **Line 506:** "Write your PR description, issue, and forum posts in your own words. Do not
  paste raw model output." Applies to forum replies and to every PR description.

### Forum discussion

[Core FR & Proposal: Workflow Test Harness & Format](https://community.n8n.io/t/core-fr-proposal-workflow-test-harness-format/306721/3)

---

## 16. Appendix: File Reference Index

Verified against n8n `2.34.0` (`9d9e9bf97e`). Paths relative to repo root.

**Mock primitives**
- `packages/core/src/execution-engine/node-execution-context/utils/request-helpers/factory.ts:69,144,195,215` — HTTP hook sites
- `packages/core/src/execution-engine/node-execution-context/utils/request-helpers/authentication.ts:41,165` — authenticated request hooks
- `packages/core/src/execution-engine/node-execution-context/node-execution-context.ts:332` — credential synthesis
- `packages/core/src/execution-engine/eval-mock-helpers.ts:27,36` — throwaway RSA key; secret-name heuristics
- `packages/core/src/execution-engine/index.ts:80` — `configureAdditionalData`
- `packages/core/src/execution-engine/workflow-execute.ts:1830,1832` — pinData substitution; runIndex 0

**Existing mock-handler plantings**
- `packages/cli/src/modules/instance-ai/eval/execution.service.ts:556,868-872` — intercepting handler; in-memory `interceptedRequests`
- `packages/cli/src/modules/instance-ai/eval/agent-execution.service.ts:315-322,538` — `EvalMockedCredentialsHelper`; `createRecordingMockHandler`
- `packages/cli/src/modules/agents/agent-runtime-instrumentation.ts:42-50` — `configureToolAdditionalData`, sub-execution propagation precedent

**Mock response behavior**
- `packages/core/src/execution-engine/index.ts:16,27` — `EvalMockHttpResponse` (no network-error field); `EvalLlmMockHandler`
- `packages/core/src/execution-engine/eval-mock-helpers.ts:201-250` — `callEvalMockHandler`; status ≥ 400 throws HTTP-library-shaped errors
- `packages/core/src/execution-engine/node-execution-context/utils/request-helpers/pagination.ts:134,141` — per-page calls go through hooked helpers
- `packages/workflow/src/jmespath-query.ts` — JMESPath, available in expressions

**Nodes used in the reference workflow**
- `packages/nodes-base/nodes/Hubspot/V2/GenericFunctions.ts:34,43` — absolute URL; `requestWithAuthentication`
- `packages/nodes-base/nodes/Hubspot/V2/HubspotV2.node.ts:2269-2297,2389-2461` — search by domain; deal update (`PUT /deals/v1/deal/{id}`)
- `packages/nodes-base/nodes/Salesforce/GenericFunctions.ts:96-140` — JWT path; relative URL

**Execution & runner**
- `packages/cli/src/workflow-runner.ts:355,356,415` — mode gate; per-run pinData; configureAdditionalData
- `packages/cli/src/workflows/workflow-execution.service.ts:155,192,229` — pinData always from saved workflow
- `packages/@n8n/api-types/src/dto/workflows/manual-run.dto.ts` — no pinData field; `schemaFor` dispatch
- `packages/cli/src/workflow-execute-additional-data.ts:560-586` — sub-execution whitelist
- `packages/workflow/src/node-helpers.ts:1265` — pinned nodes skip validation
- `packages/cli/src/workflow-helpers.ts:39` — 12 MB pinData cap

**Storage & pruning**
- `packages/workflow/src/interfaces.ts:3351` — `ITaskData`
- `packages/@n8n/config/src/configs/executions.config.ts:107-118` — prune defaults
- `packages/@n8n/db/src/repositories/execution.repository.ts:523,542,543-553,566` — prune query; annotation exemptions
- `packages/@n8n/db/src/entities/execution-data.ts` — single `text` column
- `packages/@n8n/db/src/entities/test-run.ee.ts`, `test-case-execution.ee.ts` — outlive-pruning pattern

**Redaction**
- `packages/cli/src/modules/redaction/redaction-policy.ts:30` — floor violation
- `packages/cli/src/executions/execution-redaction.ts` — interface
- `packages/cli/src/execution-lifecycle/execution-lifecycle-hooks.ts:349-377` — fail-closed push redaction
- `packages/workflow/src/interfaces.ts:2968` — `sensitiveOutputFields`

**Evaluations**
- `packages/@n8n/backend-common/src/license-state.ts:262` — license gate
- `packages/cli/src/evaluation.ee/test-runs.controller.ee.ts:225-236` — async 202
- `packages/cli/src/public-api/v1/handlers/evaluations/evaluations.handler.ts:161-163` — public API
- `packages/cli/src/evaluation.ee/test-runner/test-runner.service.ee.ts:249,273-287` — per-case pinData; queue-mode workaround
- `packages/nodes-base/nodes/Evaluation/utils/evaluationUtils.ts:301` — `checkIfEvaluating`
- `packages/@n8n/api-types/src/dto/evaluations/evaluation-config.dto.ts` — metric schemas

**Other**
- `packages/@n8n/backend-network/src/http/fake-outbound-http.ts` — `Route` shape
- `packages/@n8n/backend-network/src/ssrf/ssrf-protection.service.ts` — HTTP-scoped, not a socket backstop
- `packages/core/nodes-testing/node-test-harness.ts` — in-process harness, unshipped
- `packages/cli/package.json:283` — `yaml` dependency
- `packages/cli/src/modules/source-control.ee/types/exportable-workflow.ts` — omits pinData
- `packages/@n8n/db/src/migrations/common/1745322634000-CleanEvaluations.ts` — the removal
- `CONTRIBUTING.md:53,457,458,462,464,466,506,529` — contribution constraints (§15)
