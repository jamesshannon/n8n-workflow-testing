# Workflow testing: working notes

This branch (`workflow-testing`) is a working area for designing end-to-end testing for n8n
workflows: mock external calls at the HTTP layer, run the workflow, and assert on what it
tried to send. It is not a proposed PR. Any upstream contribution will be rebuilt as a
sequence of small PRs once the design is agreed with the n8n team.

Everything for this effort except changes to n8n's own code lives under `workflow-testing/`.
These docs are in `workflow-testing/docs/`; other artifacts, such as a CLI runner, will sit
alongside them.

Forum discussion: [Core FR & Proposal: Workflow Test Harness & Format](https://community.n8n.io/t/core-fr-proposal-workflow-test-harness-format/306721/3)

| Path | What it is |
|---|---|
| `n8n-workflow-testing-prd.md` | The design record: findings in the n8n codebase, prior art, architecture, rejected approaches, open questions |
| `reference/invoice-deal-sync/` | A reference workflow with six test cases in the draft spec format |

## Working on this branch

- **Bring in upstream changes by merging, not rebasing.** Merge n8n release tags into this
  branch; don't rebase it. Rebasing rewrites history and breaks every collaborator's local
  copy.
- File references in the PRD were checked against n8n `2.34.0` (`9d9e9bf97e`). Re-check
  them after merging a newer release.
