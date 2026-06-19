# system-autocommit

`tier: system` · `system: true` · force-attached, locked.

Owns two things the worker `-p` prompt never mentions: **git authentication
inside the ephemeral container** and an **in-session, agent-authored commit** of
the worker's code changes. It contributes:

- a **SessionStart** hook that seeds a git identity and (when present) git
  credentials;
- the **`twinkle-git`** skill (the commit recipe, available as `/twinkle-git`);
- a blocking **`Stop`** gate that refuses to end a session while tracked code is
  uncommitted.

## SessionStart — identity + credential seed

Runs once at session start, POSIX `/bin/sh`:

1. **Identity first, unconditionally.** Seeds
   `git config --global user.name "${GIT_AUTHOR_NAME:-twinkle-worker}"` and
   `user.email "${GIT_AUTHOR_EMAIL:-twinkle-worker@orchestra.local}"`. This runs
   **before** the token check so an in-session commit always has an author —
   without it `git commit` aborts with "who are you". Writes to the ephemeral
   container `~/.gitconfig` (`/home/worker/.gitconfig`, not a mount); idempotent
   per session.
2. **Credentials, if available.** If `GITHUB_TOKEN` is unset it logs a skip note
   to stderr and exits 0 (identity is still seeded). Otherwise it installs a
   credential helper that emits `username=x-access-token` / `password=$GITHUB_TOKEN`,
   rewrites `git@github.com:` → `https://github.com/`, and echoes a diagnostic
   naming the resolved `origin` remote.

**Token handling note:** the helper is single-quoted, so `${GITHUB_TOKEN}` is
expanded by git at credential time, not baked into the gitconfig at seed time.
The gitconfig is ephemeral (container-local), so the token never persists to a
mount or to the worker repo.

## `twinkle-git` skill — the commit recipe

After completing the task, in order:

1. Write `task/result.md` (see the `/write-result` skill / [system-result](../system-result/README.md)).
2. Stage code, excluding session scaffolding:
   `git add -A -- . ':(exclude).claude'`. This skips `/work/.claude/` (the
   settings/skills the platform injects and strips); `task/` and `logs/` are
   already in the worker repo's `.git/info/exclude`.
3. Commit with the task id + a one-line summary:
   `git commit -m "$(jq -r '.task_id' task/context.json): <one-line summary>"`.
   The identifier is the `.task_id` field of `task/context.json`.
4. Push is **best-effort/optional** (`git push 2>/dev/null || true`) — the
   platform backstops the push. There is **no** mandatory `git pull --rebase`
   (conflict-prone in-session, and it would fight the deterministic backstop).

If a commit is genuinely impossible (e.g. an unresolvable merge conflict), record
that in `task/result.md` rather than leaving the tree dirty.

## The Stop enforcement gate

A blocking `Stop` hook (POSIX, runs in the worktree cwd `/work`):

- Reads the Stop payload on stdin; **exits 0 immediately when
  `stop_hook_active` is true** (one-round livelock cap).
- Computes the dirty set with `git status --porcelain -- . ':(exclude).claude'`
  — the **exact same exclusion** as the `twinkle-git` skill's `git add` pathspec.
- **Exits 0 (passes) when the tree is clean.** This covers both the nothing-to-do
  case and the already-committed-but-unpushed case (an already-committed tree is
  clean to `git status`), so the gate never blocks on a tree that just needs a
  push.
- **Exits 0 when only `.claude/` is dirty** — never blocks on session
  scaffolding.
- **Exits 2 (blocks) only when tracked/untracked code outside `.claude/` is
  uncommitted**, with a message pointing at `/twinkle-git`.

It **enforces commit, not push.** Push is left to the platform.

## Relationship to `git_guard_post`

The runner's `git_guard_post` (PROCESS phase, in `runner.sh`) is the
**deterministic push backstop**: it commits anything still dirty and pushes,
recording an unpushed sentinel on failure for retry next cycle. The overlap with
this plugin is **by design**:

- When the plugin commits cleanly in-session, `git_guard_post` finds nothing to
  commit and simply owns the **push** (no-op on a clean, synced tree).
- The guard is the algorithmic safety net (it runs even if the agent crashes
  before its Stop hook); the plugin is the graceful, LLM-authored path with a
  meaningful commit message.

The plugin gate enforcing **commit only** is what keeps the two from fighting:
the guard owns the deterministic push.

## Dependencies & runtime

- `git`, `jq` — present in the worker image.
- cwd `/work`; POSIX `/bin/sh` (dash); blocking is `exit 2`.
- `stop_hook_active` caps blocking at one round.

## Testing

Tier-1 deterministic suite:
`services/controllers/runner/tests/system-autocommit.bats`. It builds real temp
git repos (with an upstream so `@{u}` resolves) and asserts the Stop gate blocks
on an uncommitted dirty tree, passes on a clean tree / `.claude/`-only-dirty tree
/ already-committed-unpushed tree / `stop_hook_active=true`; and asserts the
SessionStart hook runs cleanly under dash (no `[[` error) and seeds the fallback
identity even with `GITHUB_TOKEN` unset.

See the repo [README](../../README.md) for the two-tier model.
