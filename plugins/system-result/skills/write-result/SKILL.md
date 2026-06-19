---
name: write-result
description: "Write task/result.md with task outcome. Required — the session cannot end without a valid result.md (enforced by blocking Stop hook)."
---

Write `task/result.md` with YAML frontmatter reporting the task outcome.

## Statuses

- `status: done` — task completed successfully.
- `status: needs_human` — you need something only a human can provide (credentials, external access, approval). Add `questions:` list.
- `status: failed` — technical failure (network, filesystem, tools broken). Add `error:` field.

## Optional fields

- `summary:` — brief description of what was accomplished
- `discovered_tasks:` — follow-up work, list of `{title, description, worker}`

## Example

```yaml
status: done
summary: "Implemented authentication flow with OAuth2"
discovered_tasks:
  - title: "Add rate limiting to auth endpoint"
    description: "Current endpoint has no rate limiting, could be abused"
    worker: "site-developer"
```

## Rules

- Always write result.md as the LAST action before ending the session
- Be honest about status — `failed` is better than a false `done`
- Use `needs_human` only when you truly cannot proceed without human input
