# twinkle-flow-system

The **system / life-support** tier of the Orchestra plugin catalog. These three
plugins are the bedrock the platform cannot run without: they make sure every
worker session ends with a valid result artifact, commits its code, and starts
oriented toward the task. They are the **gold-standard** repo — the per-plugin
docs and the two-tier test template here are the pattern the core and company
tiers copy.

```toml
# flows.toml
[package]
name = "artme-dev/twinkle-flow-system"
version = "v0.1.0"

[exports]
plugins = "*"
```

## Tier & lock model

This repo is **force-attached globally and locked**. Every plugin carries
`system: true` in its `plugin.yaml`. In the Orchestra UI the force-attach preset
for this repo is non-removable and every plugin is locked-on: a workspace admin
cannot detach `twinkle-flow-system` or toggle any of its plugins off. They are
injected into **every session of every worker**, independent of which libraries a
workspace attaches.

| Plugin | Job (must produce) | Mechanism |
|---|---|---|
| [`system-result`](plugins/system-result/README.md) | `task/result.md` exists, has a valid status, and contains every declared flow output | SessionStart rules text + two blocking `Stop` gates + the `write-result` skill |
| [`system-autocommit`](plugins/system-autocommit/README.md) | Worker code changes committed in-session with a task-id message; git auth + identity seeded in-container | SessionStart cred/identity seed + the `twinkle-git` skill + a blocking enforcement `Stop` gate |
| [`system-kick-start`](plugins/system-kick-start/README.md) | The agent is oriented from the runner-assembled `task/` files | Single SessionStart narration hook (no skill) |

The worker's `claude -p` prompt is a static pointer to `./task/` — it never
mentions `result.md` or git. These plugins are therefore the **sole** in-session
owners of those behaviors. The runner owns only the *read* side (`parse_result`)
and the *deterministic backstop* (`git_guard_pre`/`git_guard_post`).

## Session-injection mechanism

How a plugin reaches a live worker session:

1. A plugin declares behavior in `plugin.yaml` under `session.hooks`
   (`SessionStart` / `Stop` entries) and `session.skills` (file references to
   `skills/<name>/SKILL.md`). The plugin **authors** the hooks — the runner is a
   pure transport (the session-injection split).
2. flows-service serves the resolved plugin view. For a workspace-scoped worker
   (`WORKSPACE` is always set), force-attach plugins resolve through the
   flows-service **`workspacePluginView`** — the union of attached libraries and
   force-attach globals. (Before the wave-1 fix, the workspace view excluded
   force-attach repos, so these system plugins silently never fired; that path is
   now fixed.)
3. The runner's `setup_plugins` (`services/controllers/runner/runner.sh`) fetches
   each plugin and writes its pieces into the worker directory:
   - `session.hooks` are converted to Claude-native format and merged into
     `/work/.claude/settings.json` under `.hooks` (arrays concatenated across
     plugins in resolution order).
   - each `session.skills` entry is written to
     `/work/.claude/skills/<plugin>-<skill>/SKILL.md` (the dir name is
     `<plugin-name>-<skill-name>`, e.g. `system-result-write-result`).
4. The worker container runs `claude -p <prompt>` with cwd `/work`, which reads
   the project-scoped `/work/.claude/settings.json` and the project skills.
5. After the session, the runner's `cleanup_plugins` strips the injected
   `.claude/` artifacts.

### Hook constraints

- Hooks run under the worker image's `/bin/sh` — **dash** (`node:22-slim` /
  Debian). Commands **must be POSIX** (no `[[ ]]`, no `(( ))`, no `<()`, no bash
  arrays). `jq` is available in the worker image.
- Plugin hook entries support **only** the keys `name`, `command`, `blocking`,
  and `message`. `PluginHookEntrySchema` is `.strict()` — any other key (e.g.
  `timeout`) fails lint. Blocking is achieved by **`exit 2`** from the command.
- The runner applies a **30s default timeout** to every plugin hook command
  (`.timeout // 30` in `convert_hooks_to_native`); it cannot be raised
  plugin-side because the schema forbids a `timeout` key.
- For `Stop` hooks, Claude Code caps blocking at **one round** via
  `stop_hook_active`: when the agent is re-prompted after a blocked Stop, the
  next Stop carries `stop_hook_active: true`. Every gate here exits 0 immediately
  in that case so a session can never live-lock.

## Canonical `task/result.md` format

All three plugins (kick-start's closing line, the `write-result` skill, and the
`system-result` gates) describe the **same** strict format. The runner's
flow-output uploader requires document frontmatter; without the fences it
silently drops every declared output, so the format is non-negotiable:

```
---
status: done
summary: One-line summary
<declared_output_key>: value      # one per name in task/expected-outputs.json, if any
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

- The file **MUST** begin with `---` on line 1 and close the frontmatter with a
  second `---`. Frontmatter keys sit at **column 0**.
- **Status vocabulary** (the full runner-accepted set):
  `done`, `done_with_concerns`, `needs_human`, `blocked`, `needs_context`,
  `failed`.
- Every name listed in `task/expected-outputs.json` **MUST** appear as a
  top-level frontmatter key (required for the runner's flow-output upload).

## Testing

Every enforcing behavior here is **non-deterministic to observe live** — a
healthy agent writes `result.md` and commits *before* its first `Stop` hook
fires, so a live run alone never exercises the gates. Testing is therefore two
tiers.

### Tier 1 — deterministic hook-command harness (primary proof, CI-runnable)

Lifts each plugin's hook `command` string straight out of `plugin.yaml` and runs
it under **dash** (matching the worker image) against crafted temp `task/`
fixtures, asserting exit code (0 = pass-through, 2 = blocking) + stderr/stdout.
No live system required.

- Harness: `services/controllers/runner/tests/lib/plugin-hook-harness.bash`
  (in the twinkle monorepo).
- Suites: `services/controllers/runner/tests/system-result.bats`,
  `system-autocommit.bats`, `system-kick-start.bats`.

The harness resolves these plugin sources via `SYSTEM_PLUGINS_DIR` (default: the
sibling `flow-build/twinkle-flow-system/plugins` checkout). Run with `bats`:

```sh
bats services/controllers/runner/tests/system-result.bats
```

### Tier 2 — live sandbox lane (integration confidence)

`tests/scenarios/harness/system-plugins-live.test.mjs` (cloned from
`l2-execution.test.mjs`): seed a sandbox worker, dispatch one uniquely-marked
labelled task, start the worker, and assert the observables — the JSONL
`SessionStart` hook entry, a terminal Linear state, a moved remote HEAD (push
backstop), and the structural precondition that the host
`workers/sandbox/<w>/.claude/settings.json` contains all three plugins'
`SessionStart`/`Stop` entries. Some behaviors are observation-limited (a healthy
agent never trips a gate); the deterministic proof lives in Tier 1.

> The cross-plugin `Stop`-hook ordering / independence contract is documented
> separately (it depends on a live multi-hook exit-2 observation).
