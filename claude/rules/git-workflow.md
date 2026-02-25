# Git Workflow Rules

## Branch Protection

**Never push directly to `master` or `main`.** All changes must go through a feature branch and PR.

**Before writing any code**, check the current branch. If on `master` or `main`, create and switch to a new feature branch **before making any changes**:
```bash
git branch --show-current  # must NOT be master or main
git checkout -b <descriptive-branch-name>  # create branch BEFORE any edits
```

This applies to all work — new features, bug fixes, refactors, config changes. Never start editing files while on `master`/`main`.

## Branch Naming

Use kebab-case with a short prefix that matches the type of change:
- `feat/<description>` — new feature
- `fix/<description>` — bug fix
- `chore/<description>` — tooling, config, deps
- `refactor/<description>` — refactoring without behavior change

## Commit Flow

The following applies to every push — whether it is the first push on a new branch or an additional commit on a branch that already has an open PR.

1. Make changes on the feature branch.
2. Stage specific files (never `git add -A` blindly).
3. Commit with a clear message explaining the **why**.
4. **Before pushing**, run the `code-reviewer` agent (via the Task tool with `subagent_type: "code-reviewer"`) against all files staged in the commit. All Critical issues surfaced by the review must be resolved before pushing. Warnings should be addressed where practical.
5. Push with `-u` to set upstream: `git push -u origin <branch>`.

## PR Creation

Use `gh pr create` to open PRs. Always include:
- A short title (under 70 characters)
- A body with a brief summary and test plan

Let CI run all non-e2e tests (unit, lint, type-check) on the PR — do not run these locally.

If a change touches flows covered by e2e tests (auth, dashboard, full upload flow), prompt the user to run `pnpm test:e2e` locally against their dev stack **before** pushing to the PR.

## Prohibited and Restricted Operations

### Strictly Forbidden — No Exceptions

AI agents must **never** rebase, amend, squash, or otherwise rewrite git history — whether on local or remote branches. Explicit user approval does not override these prohibitions. This includes:

- `git rebase` (interactive or otherwise), on any branch at any state
- `git commit --amend` on any commit that has already been pushed
- `git reset` (any mode: `--hard`, `--soft`, `--mixed`) to remove or undo commits
- `git push --force` or `git push --force-with-lease`
- Any other operation that changes the SHA of an existing commit

If a situation seems to call for history rewriting, stop, explain the situation to the user, and ask them to perform the operation manually.

### Requires Explicit User Approval

- Deleting a branch
- Merging or closing a PR
