# Stop-hook coexistence & independence contract

The force-attached plugins contribute **multiple `Stop` hooks** to every worker
session. This document is the contract that keeps them safe together, and the
guarantee any future plugin adding a `Stop` hook must uphold.

With **both** the system tier and the core tier force-attached (the live
default), a worker session carries **~6 blocking `Stop` gates**:

| # | Source | Tier | Gate |
|---|---|---|---|
| 1 | `system-result` | system | `task/result.md` present + valid status |
| 2 | `system-result` | system | declared flow outputs present in result.md frontmatter |
| 3 | `system-autocommit` | system | tracked code committed in-session |
| 4 | `core-memory` | core | `task/journal.md` present (no-ops when journal-mcp is unwired) |
| 5 | `core-state` | core | `state.md` present + `<!-- last_task: ... -->` first-line marker |
| 6 | `core-summary` | core | `task/summary.yaml` declares `type:` + `summary:` |

(System gates are locked-on; core gates are toggleable per plugin, so a workspace
may run with fewer.)

## How Stop hooks are merged into a session

At PREPARE the runner's `setup_plugins` resolves all plugins for the session and concatenates
each plugin's `session.hooks[event]` arrays into the worker's `/work/.claude/settings.json`
under `.hooks.<event>` (task-level hooks first, then plugin hooks, in plugin-resolution order).
`convert_hooks_to_native` wraps each legacy `{name, command, blocking, message}` entry into
Claude-native `{hooks:[{type:command, command, timeout}]}` and **drops the source `name`** — so
in the live file a hook's only stable identity is its command body. There is **no ordering key**:
the relative order of two plugins' `Stop` hooks is just their resolution order.

A worker with no own plugins (the common system-only case) gets, for `Stop`:

| Source | Gate | Blocks (exit 2) when |
|---|---|---|
| `system-result` | result presence + valid status | `task/result.md` is missing, or its `status:` is not in the accepted set |
| `system-result` | declared flow outputs present | a name in `task/expected-outputs.json` is absent from the result.md frontmatter |
| `system-autocommit` | in-session commit enforced | there is uncommitted **tracked** code (excluding `.claude/`) |

## The two invariants that make order not matter

**1. One-round blocking cap.** Claude Code sets `stop_hook_active = true` once a `Stop` hook has
blocked and the agent re-attempts to stop. Every gate here checks it **first** and `exit 0` when
true. So the gates get exactly **one** enforced round across the whole session. Consequences:

- A gate must report **everything** it knows is wrong in that one round — e.g. the flow-outputs
  gate accumulates and names **all** missing outputs (`missing required output(s): a, b, c`), not
  just the first, because there is no guaranteed second round to catch the rest.
- A gate must be **idempotent** and must never block on a state it can't help the agent fix.

**2. Independence.** Each gate blocks on a **disjoint** condition:

- result-gate ⇢ only about `task/result.md` (presence / status / declared outputs).
- autocommit-gate ⇢ only about uncommitted tracked code (excluding `.claude/`, the same pathspec
  the `twinkle-git` skill stages with).
- journal-gate (core) ⇢ only about `task/journal.md` (and only when journal-mcp is wired).
- state-gate (core) ⇢ only about `state.md` + its `<!-- last_task: ... -->` first-line marker.
- summary-gate (core) ⇢ only about `task/summary.yaml`'s `type:` + `summary:` keys.

Each of the six owns a **distinct artifact**, so no two ever contend. Because the conditions don't
overlap, the **merge order is immaterial on a healthy run**: an agent that writes a valid
`result.md`, commits its code, and writes its journal / state / summary artifacts satisfies every
gate regardless of which fires first.

## Multi-Stop aggregation — CONFIRMED FROM SOURCE

**This is the load-bearing fact the whole 6-gate set rests on, and it is settled
from source — claude-code 2.1.92 (the worker image's bundled CLI).** Reading the
2.1.92 `cli.js`:

> Claude Code runs **all** `Stop` hooks registered for the event and **aggregates
> every exit-2 blocking message** into the **single** enforced re-prompt round. It
> is a **concurrent generator-merge** — there is **no abort-on-first**. The
> feared "only one artifact enforced per session" does **not** occur: each gate's
> stderr is carried back to the model together.

`stop_hook_active` then caps the session at **exactly one** enforced round — but
that one round carries **every** gate's complaint at once. So with all six gates
firing on a non-terminal session, the agent is re-prompted once with the union of
all six blocking messages, and is expected to satisfy all of them before its next
(now `stop_hook_active: true`) Stop passes through.

This **upgrades** the previously observation-limited note: per-`Stop`-hook
exit codes are still not on any gateway-observable surface (the stream-json JSONL
omits per-hook telemetry and the container is removed on stop), but the
aggregation behavior is no longer inferred — it is read from source and pinned by
a test (below).

### The aggregation capstone (pins the behavior against image bumps)

`services/controllers/runner/tests/stop-aggregation.bats` (in the twinkle
monorepo) is a docker-based Tier-1 capstone that converts the source-read into a
binary test: it writes a synthetic `/work/.claude/settings.json` with **three**
trivial, independent exit-2 `Stop` hooks (distinct stderr markers), runs
`orchestra-worker-image` with `claude -p` on a one-line prompt exactly as the
runner does, and asserts **all three** markers appear in one turn. If a future
worker-image bump changes aggregation (e.g. to abort-on-first), the test goes red
in CI before the broken image ships. It SKIPs (never false-greens) without a
Claude token or the built image.

## Observed behavior (Tier-2 live, 2026-06-20)

A real `sandbox` worker run (`tests/scenarios/harness/system-plugins-live.test.mjs`) confirmed the
three system `Stop` gates coexist (`Plugin: Stop hooks count=3`) and the session ended cleanly with a
valid `result.md` **and** an in-session commit — i.e. no gate left the task non-terminal or the tree
uncommitted. The autocommit gate was observed to `exit 2` on uncommitted code and the agent committed
in response (then the runner `git_guard_post` backstopped the push). The core-tier live lane
(`core-plugins-live.test.mjs`) extends this to the full ~6-gate set (asserting mid-run
`.hooks.Stop` ≥ 6 + a clean terminal).

## Contract for any future Stop hook (system or otherwise)

1. **Honor `stop_hook_active`** — read stdin, `exit 0` when it is `true`. Never reintroduce a
   multi-round loop.
2. **Stay independent** — block only on a condition disjoint from every other gate's, so merge order
   stays irrelevant. If you cannot, you must define and enforce an explicit ordering in
   `setup_plugins` (none exists today, by design).
3. **Accumulate within your own round** — report every problem you can detect in the single enforced
   round; do not rely on a second.
4. **POSIX `/bin/sh` (dash)** — no `[[`, no process substitution; ensure `exit 2` propagates from
   any loop (a `cmd | while` subshell would swallow it — use a heredoc/redirect that keeps the loop
   in the current shell).
5. **Block (exit 2) only on a state the agent can fix in-session**, and never on a clean/no-op state.
