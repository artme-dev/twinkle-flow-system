# system-kick-start

`tier: system` · `system: true` · force-attached, locked.

Orients the agent at the start of every session. It reads the files the runner
assembled under `task/` and narrates **what to work on** and **what "done" looks
like**. It is a single **SessionStart** hook — **no skill**, no Stop gate.

## Inputs & shapes

The hook reads (all relative to cwd `/work`):

| File | Shape | Rendered as |
|---|---|---|
| `task/flow-prompt.md` | markdown | `## Your Task` + the prompt verbatim (preferred when present) |
| `task/context.json` | `{ title, description, feedback }` | fallback `## Your Task` (`**title**` + `description`) when no flow-prompt.md; also the source of feedback |
| `task/context/<dep>.md` | markdown files | `## Outputs from Previous Tasks`, one `### <dep>` section per file |
| `task/expected-outputs.json` | JSON **array** of names | `## Required Outputs` — the keys the `result.md` frontmatter must include |

`context.json.feedback` is a JSON **array** of `{ author, date, body }` objects
(a `type=="string"` back-compat branch is also handled). Populated feedback
renders a `## Feedback from Prior Attempts` section with
`- **author** (date): body` bullet lines; an empty array `[]` emits **no**
section. An empty Linear title falls back to `**Task**` (not `****`). When neither
`flow-prompt.md` nor `context.json` exists, it emits a one-line note before the
closing instruction. If `task/` itself is absent, the hook exits 0 with no
output.

## Output contract

The hook writes its narration to **stdout**, which Claude Code injects into the
session as SessionStart context. It always closes with:

> `Work in ./task/. Write your outcome to task/result.md before ending the
> session.`

It produces no Stop gate and stages nothing — it is pure orientation.

## Known limitation — 30s hook timeout (KICK-4)

The runner applies a **30-second default timeout** to every plugin hook command
(`.timeout // 30` in `convert_hooks_to_native`). This **cannot be raised
plugin-side**: `PluginHookEntrySchema` is `.strict()` and permits only the keys
`name`, `command`, `blocking`, `message` — adding a `timeout` key fails lint. (The
`.timeout` passthrough only applies to *task*-level hooks, not plugin hooks.)

Consequence: if the narration takes longer than 30s — a very large `task/context/`
set, or a `jq` parse that stalls/errors — the hook is killed and **silently
no-ops** (no orientation injected; the session still proceeds). A real fix would
be a schema change, out of scope for this plugin.

## Dependencies & runtime

- `jq` (rendering `context.json` and `expected-outputs.json`) — present in the
  worker image.
- cwd `/work`; POSIX `/bin/sh` (dash); exits 0 (non-blocking).

## Testing

Tier-1 deterministic suite:
`services/controllers/runner/tests/system-kick-start.bats`. It asserts an empty
feedback array emits no Feedback section; a populated array renders
`- **author** (date): body`; an empty title renders `**Task**` not `****`; a
brief-less `task/` emits the guard note; and an absent `task/` exits 0 with no
narration. (KICK-4 is documented, not tested — it is unfixable plugin-side.)

See the repo [README](../../README.md) for the two-tier model.
