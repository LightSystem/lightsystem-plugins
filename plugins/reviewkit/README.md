# reviewkit

Two skills for reviewing a change with more than one detector and trusting none of them.

| skill | what it does |
|---|---|
| `/reviewkit:review-round` | Runs all three lanes — `/simplify`, the CodeRabbit CLI, IntelliJ's inspection engine — concurrently over one resolved scope, then merges, deduplicates and verifies every finding before acting on it. |
| `/reviewkit:coderabbit-triage` | The CodeRabbit lane on its own, plus the verification altitudes, the FIX/SKIP/FLAG classification and the report piles that `review-round` defers to. |

`review-round` reads `coderabbit-triage` as a document even when that lane cannot run, so install
both or neither. Each lane is independently optional: a missing one is reported, never silently
dropped.

## Requirements

Nothing is mandatory; each unmet requirement costs one lane.

- **CodeRabbit CLI** (`cr`/`coderabbit`) on `PATH` and authenticated (`cr auth login`), and `jq`.
  No GitHub remote and no `gh` — the review runs against the working tree, not a pull request.
- **IntelliJ IDEA** with its MCP server connected (`mcp__idea__*`).
- `/simplify` ships inside the `claude` executable and needs nothing.

## What a repository supplies

Both skills read `REPO-NOTES.md` at the repository root, and both work without it — every
reference carries a fallback, at the cost of rediscovering the same facts every round. It earns
its keep by recording what a reviewer cannot derive from the source:

- the validation commands, and which of them regenerate inputs, in what order
- how this build reports a stale green
- findings already judged noise, and why
- paths excluded from the IDE project
- configuration that enumerates paths, so a rename can be checked against it

Step 5 of `coderabbit-triage` also writes durable conventions back into `.coderabbit.yaml`
`path_instructions`, which is where a project-wide review rule belongs; per-repository facts go
to `REPO-NOTES.md` instead.

The base branch is assumed to be `main`; a repository that calls it something else edits the
`--base` flags in `coderabbit-triage`'s **User Input**.
