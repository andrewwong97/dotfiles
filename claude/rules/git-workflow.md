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

Branch names must be descriptive of the work being done. If you are working in a worktree whose name is not descriptive (e.g. an auto-generated name), **rename the branch** to something meaningful before pushing:
```bash
git branch -m <old-name> <type>/<descriptive-name>
```

## Beads-Driven Workflow

When work is tracked via beads (`bd`), **each beads task gets its own branch and PR**. The full lifecycle for a single task:

1. `bd ready` — find the next unblocked task.
2. `bd update <id> --status=in_progress` — claim it.
3. `git checkout master && git pull` — start from latest master.
4. `git checkout -b <type>/<task-description>` — branch name matches the task scope.
5. Implement, commit, run code review, push, open PR (reference the beads ID in the commit and PR body).
6. **Do NOT close the beads task yet** — it stays `in_progress` until the PR is merged.
7. After the PR is merged: `bd close <id> --reason "Merged in PR #N"`.
8. `bd ready` — check what's unblocked next.

Each PR should be independently mergeable. If tasks have dependencies in beads, the later branch is created off master **after** the dependency's PR is merged (or rebased by the user).

## Commit Flow

The following applies to every push — whether it is the first push on a new branch or an additional commit on a branch that already has an open PR.

1. Make changes on the feature branch.
2. **Run related tests** before committing (code changes only — docs, config, and other non-code changes skip this step). For backend changes: `cd backend && source venv/bin/activate && python -m pytest <APP>/tests/ -x -q`. For frontend changes: run the relevant test files with `pnpm test`. Fix any failures before proceeding.
3. Stage specific files (never `git add -A` blindly).
4. Commit with a clear message explaining the **why**.
5. **Before pushing**, run the `code-reviewer` agent (via the Agent tool with `subagent_type: "code-reviewer"`) against all files staged in the commit. All Critical issues surfaced by the review must be resolved before pushing. Warnings should be addressed where practical.
6. Push with `-u` to set upstream: `git push -u origin <branch>`.

## PR Creation

**After every push**, open or update a PR for the branch:
- If no PR exists for the branch, create one with `gh pr create`.
- If a PR already exists, it updates automatically with the new commits — no action needed.

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
