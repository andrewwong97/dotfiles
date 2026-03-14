Run the code-reviewer agent against the current uncommitted changes.

## Steps

1. Run `git diff` to identify what files have changed.
2. If there are no changes, report that there is nothing to review and stop.
3. Invoke the `code-reviewer` subagent (via the Agent tool with `subagent_type: "code-reviewer"`) to review all changed files.
4. Report the results, grouped by priority level (Critical, Warnings, Suggestions).
5. If there are Critical issues, list them clearly and note they must be fixed before committing.
