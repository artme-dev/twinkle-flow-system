# twinkle-git — commit your work before ending the session

Worker code changes must be **committed in-session**. The platform deterministically
backstops the *push*, so pushing here is best-effort/optional — but committing is your
job, and the session will be blocked from ending while tracked code is uncommitted.

After completing the task, in this order:

1. **Write `task/result.md`** (use the `/write-result` skill for the format).

2. **Stage your code changes, excluding session scaffolding:**

   ```sh
   git add -A -- . ':(exclude).claude'
   ```

   This stages all changed files but skips `.claude/` (live-session settings/skills
   that the platform injects and strips). `task/` and `logs/` are already ignored on
   the worker repo, so they are never staged.

3. **Commit with a message carrying the task id and a one-line summary:**

   ```sh
   git commit -m "$(jq -r '.task_id' task/context.json): <one-line summary of what you did>"
   ```

   The task identifier is the `.task_id` field in `task/context.json`. Replace
   `<one-line summary ...>` with a concise description of the change.

4. **Push (best-effort, optional):**

   ```sh
   git push 2>/dev/null || true
   ```

   If the push fails (no credentials, offline, non-fast-forward), do **not** block on
   it — the platform's git backstop will push the commit. Do not run `git pull --rebase`
   as a mandatory step; it is conflict-prone in-session and would fight the backstop.

If anything prevents you from committing (e.g. a merge conflict you cannot resolve),
record the situation in `task/result.md` instead of leaving the tree dirty.
