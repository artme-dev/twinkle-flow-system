# system-result

`tier: system` · `system: true` · force-attached, locked.

Sole producer and enforcer of `task/result.md`. The worker `-p` prompt never asks
for `result.md`, so this plugin is the **only** thing that makes a session record
its outcome. It contributes:

- a **SessionStart** rules hook that tells the agent the result lifecycle;
- the **`write-result`** skill (the format reference, available as
  `/write-result`);
- **two blocking `Stop` gates** that refuse to end a session without a valid,
  complete `result.md`.

## Canonical format

`task/result.md` is strict YAML frontmatter (the single source of truth shared
across all three system plugins):

```
---
status: done
summary: One-line summary
<declared_output_key>: value      # one per name in task/expected-outputs.json
concerns:                          # optional, for done_with_concerns
  - ...
context_needed: ...                # optional, for needs_context
discovered_tasks:
  - title: ...
    description: ...
    worker: ...
---
free-form markdown body (optional)
```

- MUST begin with `---` on line 1 and close the frontmatter with a second `---`.
- Keys sit at **column 0**.
- Every name in `task/expected-outputs.json` MUST be a top-level frontmatter key.

The runner's flow-output uploader extracts the frontmatter with
`sed -n '/^---$/,/^---$/p' task/result.md | sed '1d;$d'`; without the fences it
silently drops everything. The `write-result` skill teaches exactly this fenced
example (not a ` ```yaml ``` ` block).

## Status vocabulary

The full runner-accepted set. Pick the one that is honest:

| status | meaning | partner field |
|---|---|---|
| `done` | completed successfully | `summary:` |
| `done_with_concerns` | completed, but with caveats worth flagging | `concerns:` (list) |
| `needs_human` | needs something only a human can provide (credentials, external access, approval) | `questions:` (list) |
| `blocked` | cannot proceed, but not a technical failure (unmet precondition, pending decision) | `summary:` (why) |
| `needs_context` | needs an unavailable dependency / upstream output | `context_needed:` (scalar) |
| `failed` | technical failure (network, filesystem, broken tools) | `error:` |

## The two Stop gates

Both gates run under dash, read the Stop payload on stdin, and **exit 0
immediately when `stop_hook_active` is true** (one-round blocking cap — a
re-prompted Stop never blocks again).

1. **Result exists + valid status.** Blocks (`exit 2`) if `task/result.md` is
   missing, or if its `status:` is not one of the six values above. The
   error names the full allowlist and points at `/write-result`.
2. **Declared flow outputs present.** No-ops (exit 0) when `result.md` or
   `task/expected-outputs.json` is absent. Otherwise it slices the frontmatter
   with the runner's exact extract idiom, then checks that every declared output
   name appears as a `^<name>:` frontmatter key. Missing names are accumulated
   and **all** reported in a single blocking message (the one-round Stop cap
   means naming only the last would let the others slip through). The
   output-name iteration is space-safe (a declared key like `out b` is matched
   whole, not word-split).

## The `expected-outputs.json` contract

`task/expected-outputs.json` is a JSON array of output names, e.g.
`["artifact_path","report_url"]`. The **runner writes it only when the current
flow task declares `outputs`**; if the task declares none, the runner removes the
file. So:

> **absent `expected-outputs.json` == no outputs required == gate 2 no-ops.**

The gate never invents requirements; it only enforces what the flow contract
declared.

## Dependencies & runtime

- `jq` (parsing the Stop stdin and the outputs array) — present in the worker
  image.
- cwd is `/work`; the gates read `task/result.md` / `task/expected-outputs.json`
  relative to it.
- POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.

## Testing

Tier-1 deterministic suite (run under dash):
`services/controllers/runner/tests/system-result.bats`. It extracts each `Stop`
command from `plugin.yaml` and asserts:

- each of the six statuses passes; an invalid status blocks and names the
  allowlist;
- a fenced result with all declared outputs passes; a missing output blocks and
  is named; multiple missing outputs are **all** named in one block;
- a space-containing output name (`out b`) present in the frontmatter passes.

See the repo [README](../../README.md) for the two-tier model.
