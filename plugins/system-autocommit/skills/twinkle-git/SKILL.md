After completing the task and writing task/result.md, perform git operations:
1. git add all changed files (excluding task/ and .claude/ directories)
2. git commit with a message containing the task identifier from task/context.json
3. git pull --rebase origin <current-branch>
4. git push

If conflicts occur during rebase, resolve them or report in result.md.
