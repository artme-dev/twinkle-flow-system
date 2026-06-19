---
name: write-result
description: "Write task/result.md with task outcome. Required — the session cannot end without a valid result.md (enforced by blocking Stop hook)."
---

Write `task/result.md` with YAML frontmatter reporting the task outcome.

## Statuses

- `status: done` — task completed successfully.
- `status: done_with_concerns` — completed, but with caveats worth flagging. Add a `concerns:` list.
- `status: needs_human` — you need something only a human can provide (credentials, external access, approval). Add `questions:` list.
- `status: blocked` — you cannot proceed, but it is not a technical failure (e.g. a precondition is unmet, a decision is pending). Add `summary:` explaining why.
- `status: needs_context` — you need a dependency or upstream output that is not available. Add `context_needed:` scalar describing what is missing.
- `status: failed` — technical failure (network, filesystem, tools broken). Add `error:` field.

## Optional fields

- `summary:` — brief description of what was accomplished
- `concerns:` — list of caveats (use with `done_with_concerns`)
- `context_needed:` — scalar describing the missing dependency (use with `needs_context`)
- `discovered_tasks:` — follow-up work, list of `{title, description, worker}`

## Example

result.md must begin with a `---` frontmatter fence on line 1. Keys sit at column 0; the closing `---` ends the frontmatter; any free-form markdown body comes after it:

---
status: done
summary: Implemented authentication flow with OAuth2
artifact_path: ./src/auth/oauth.ts
discovered_tasks:
  - title: Add rate limiting to auth endpoint
    description: Current endpoint has no rate limiting, could be abused
    worker: site-developer
---

Free-form markdown body goes here (optional).

## Rules

- result.md MUST begin with `---` on line 1 and close the frontmatter with `---`; every name in `task/expected-outputs.json` MUST appear as a top-level frontmatter key.
- Always write result.md as the LAST action before ending the session.
- Be honest about status — `failed` is better than a false `done`.
- Use `needs_human` only when you truly cannot proceed without human input.
