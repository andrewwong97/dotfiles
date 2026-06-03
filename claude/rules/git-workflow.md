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

Branch names must be descriptive of the work being done. If you are working in a worktree whose name is not descriptive (e.g. an auto-generated name), **rename the branch** to something meaningful before pushing:
```bash
git branch -m <old-name> <type>/<descriptive-name>
```

## Commit Flow

The following applies to every push — whether it is the first push on a new branch or an additional commit on a branch that already has an open PR.

1. Make changes on the feature branch.
2. **Run related tests** before committing (code changes only — docs, config, and other non-code changes skip this step). For backend changes: `cd backend && source venv/bin/activate && python -m pytest <APP>/tests/ -x -q`. For frontend changes: run the relevant test files with `pnpm test`. Fix any failures before proceeding.
3. Stage specific files (never `git add -A` blindly).
4. Commit with a clear message explaining the **why**.
5. **Before pushing**, run the `code-reviewer` agent (via the Agent tool with `subagent_type: "code-reviewer"`) against all files staged in the commit. All Critical issues surfaced by the review must be resolved before pushing. Warnings should be addressed where practical.
6. Push with `-u` to set upstream: `git push -u origin <branch>`. See **Push Cadence** — push promptly on the *first* push, but hold off on subsequent pushes until you've consulted the user.

## Push Cadence

The **first** push on a new branch should happen promptly: it triggers CI/CD and opens the PR, surfacing pipeline feedback early. Don't sit on the initial push.

**After that, do not eagerly push every commit.** Let commits accumulate locally and **consult the user before each subsequent push.** This leaves a window to consolidate similar or fix-up commits — e.g. three iterations on the same file that should be one commit — before they reach the remote.

The AI must never rewrite history itself (see Prohibited Operations), so any squash/rebase in that window is **performed by the user**: the AI's job is to hold off pushing, point out which commits could be squashed, and wait for direction. Consolidating *local, unpushed* commits is what keeps this safe — it needs no force-push. Once commits are pushed, squashing them would require forbidden history rewriting, which is exactly why the consolidation must happen **before** the next push.

## PR Creation

**After every push**, open or update a PR for the branch:
- If no PR exists for the branch, create one with `gh pr create`.
- If a PR already exists, it updates automatically with the new commits — no action needed.

Use `gh pr create` to open PRs. Always include:
- A short title (under 70 characters)
- A body with a brief summary and test plan

Let CI run all non-e2e tests (unit, lint, type-check) on the PR — do not run these locally.

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
