# Invoice reference: execution report on n8n 2.41.7

Recorded on 2026-10-07. The six reference cases passed 57 checks.
The runs used **n8n 2.41.7**, its shipped **n8n-core 2.41.5**, and reference commit
[`2298c0aefd72de1d5fbb235cd16d64543fafd16a`](https://github.com/jamesshannon/n8n-workflow-testing/tree/2298c0aefd72de1d5fbb235cd16d64543fafd16a/workflow-testing/docs/reference/invoice-deal-sync).

The [execution pack][pack] and [recorded assertions and requests][results] are fixed
at n8n-check commit `9bf6d2c26c89b94f37ae6667135c43783d67b43b`.
The pack is an experiment for this reference and its draft test format.

## Results

| Case | Engine status | HTTP calls | Checks passed |
|---|---|---:|---:|
| 1. Eligible invoices | success | 5 | 15/15 |
| 2. Pagination and filter decoys | success | 8 | 16/16 |
| 3. Missing company | success | 4 | 7/7 |
| 4. Empty page | success | 1 | 5/5 |
| 5. Billing 500, then success | success | 6 | 4/4 |
| 6. Billing 500 after all retries | success | 4 | 10/10 |
| Added case: second deal write returns 500 | error | 5 | 13/13 |
| Negative control: `amount_due` changed to `amountDue` | success | 5 | 13/15 |

The added case captures partial progress at the mocked HTTP boundary. The PUT for
deal 9001 receives success. The PUT for deal 9002 receives 500. `Audit Log` stays
unexecuted. The case expects an engine error, so its contract passes with exit 0.

The negative control changes the amount expression in a separate workflow copy.
It runs case 1. Both deal URLs remain correct. Both PUT bodies contain
`amount: "NaN"`. The two body checks fail, and the checker exits 1.

## Answers to the README's runtime questions

1. **Slack channel:** the Slack node makes one POST to
   `https://slack.com/api/chat.postMessage`. Its body contains
   `channel: "#billing-alerts"`.
2. **Fetch error output:** after three 500 responses, `main[1]` contains the
   original trigger body and an error. The Slack expression reads the error
   message. Its text starts with
   `Invoice sync failed: Request failed with status code 500`.
3. **Merge dotted path:** `properties.domain.value` matches `customer_domain`.
   Case 2 sends updates for invoices `inv-1001`, `inv-1002`, and `inv-1006`.
   Their amounts are `1250.00`, `480.00`, and `990.00`.
4. **Filter missing deal ID:** `inv-1005` goes to the rejected output. The accepted
   IDs in case 2 are exactly `inv-1001`, `inv-1002`, and `inv-1006`.

Elapsed retry timing remains a separate measurement. These results record request
counts, node outputs, and assertion outcomes.

## Execution method and scope

The adapter uses the standard CLI initialization and the official JavaScript task
runner. It calls `WorkflowExecute` in `evaluation` mode. HTTP Request, HubSpot,
and Slack execute their original node code through `evalLlmMockHandler`.
The 13 nodes, connections, URL parameters, and auth selections stay in place.

The webhook input and Postgres output use the declared pins: `Invoice Sync Webhook`
and `Audit Log`. `EvalMockedCredentialsHelper` supplies synthetic credentials.
Service responses come from the reference fixtures. JSON responses default to
`content-type: application/json`. Case headers can override that value.
The checker uses n8n's `Expression` engine and supplies `$request` for assertions.
Missing mocks and exhausted response sequences cause a contract failure.

This scope covers the pinned input, native node execution, captured HTTP calls,
and the pinned audit output. Live webhook delivery, database INSERT behavior,
live authentication, transport timeouts, and other runtime versions each require
separate checks.

## Reproduce

Check out n8n-check at `9bf6d2c26c89b94f37ae6667135c43783d67b43b`.
Use Docker Linux containers and a shell.

```sh
cd experiments/native-invoice-reference
sh run.sh               # Six original cases; exit 0
sh run.sh deal-failure  # Expected engine error; exit 0
sh run.sh negative      # Two failed amount checks; exit 1
```

The first run downloads the n8n image and nine reference files at the source
commit above. `.source/source.json` records their hashes. The execution container
uses `--network none`. Results are saved under `results/<mode>/`.

The table above comes from completed local runs in a loopback-only Linux
namespace. [GitHub Actions run 37590485838][action] also passed the Docker run for
all three modes. Its artifacts contain the full execution records.

James Shannon authored the workflow, six cases, and fixture bodies. Blucca's
autonomous AI engineering agents (GPT-6 Astra) built the adapter, added case,
negative control, and execution report.

[pack]: https://github.com/blucca/n8n-check/tree/9bf6d2c26c89b94f37ae6667135c43783d67b43b/experiments/native-invoice-reference
[results]: https://github.com/blucca/n8n-check/blob/9bf6d2c26c89b94f37ae6667135c43783d67b43b/experiments/native-invoice-reference/observed-results.json
[action]: https://github.com/blucca/n8n-check/actions/runs/37590485838
