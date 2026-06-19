# Stop-hook coexistence & independence contract (system tier)

The three force-attached system plugins contribute **multiple `Stop` hooks** to every
worker session. This document is the contract that keeps them safe together, and the
guarantee any future plugin adding a `Stop` hook must uphold.

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

Because the conditions don't overlap, the **merge order is immaterial on a healthy run**: an agent
that writes a valid `result.md` and commits its code satisfies both regardless of which fires
first. The gates never contend for the same artifact.

## Observed behavior (Tier-2 live, 2026-06-20)

A real `sandbox` worker run (`tests/scenarios/harness/system-plugins-live.test.mjs`) confirmed all
three `Stop` gates coexist (`Plugin: Stop hooks count=3`) and the session ended cleanly with a valid
`result.md` **and** an in-session commit — i.e. no gate left the task non-terminal or the tree
uncommitted. The autocommit gate was observed to `exit 2` on uncommitted code and the agent committed
in response (then the runner `git_guard_post` backstopped the push).

**Observation limit (honest):** per-`Stop`-hook exit codes and the exact intra-session
ordering/interleaving of the gates are **not** on any gateway-observable surface — `Stop`-hook
telemetry is absent from the worker stream-json JSONL, and the worker container is removed on stop,
so no per-hook stderr survives. Coexistence (settings.json + the PREPARE count) and the net outcome
(clean terminal + commit) are determinable; the exit-2/retry sequence is not.

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
