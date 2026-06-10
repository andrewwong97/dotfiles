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
2. **Run related tests** before committing (code/env-impacting changes only — docs and other non-code changes skip this step). Fix any failures before proceeding.
3. Stage specific files (never `git add -A` blindly).
4. Commit with a clear message.
5. **Before pushing**, run the `code-reviewer` agent (via the Agent tool with `subagent_type: "code-reviewer"`) against all files staged in the commit. All Critical issues surfaced by the review must be resolved before pushing. Warnings should be addressed where practical.
6. Push with `-u` to set upstream: `git push -u origin <branch>`. See **Push Cadence** — push promptly on the *first* push, but hold off on subsequent pushes until you've consulted the user.
7. **After every push, surface the PR link in your reply** (see **PR Creation**) — even when no other summary is warranted.

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

**Write PR descriptions for a reader without code context.** Describe behavior and user-visible impact in plain language. Avoid code references — symbol names, file paths, function/variable names, flags, internal identifiers — unless one is genuinely necessary for the reader to understand the change, and then keep it minimal. Prefer "behind a feature flag, off by default" over naming the flag; prefer "the saved-drafts list" over naming the query. The goal is a description a reviewer or stakeholder can understand without opening the diff.

**Structure with lists, sometimes tables.** When describing a series of steps or a set of related points, use lists rather than prose paragraphs — ordered lists for sequential steps (where order matters), unordered lists for non-sequential points. Tables are fine when the content genuinely has columns (e.g. before/after, option/effect), but favor the simplest structure that fits — reach for a table only when a list can't capture the relationship. This keeps the description scannable.

**Document new database fields (only when the change touches the database).** If the PR adds columns/fields to a table or schema, include a dedicated section with a table of the new fields — one row per field, columns for the table/model it's on, the field name, its type (and whether nullable/optional), and a plain-language description of what it stores. Omit this section entirely when the PR makes no database changes.

| Table | Field | Type | Description |
| --- | --- | --- | --- |
| `<table>` | `<field>` | `<type>`, nullable | What it holds and why |

**Always provide the PR link after every push.** Whether the push created the PR or just added commits to an existing one, end your reply with the PR URL so it's one click away. Get it from the `gh pr create` output, or `gh pr view <branch> --json url -q .url` for an existing PR. This applies to every push, not just the first.

**State the top 3 assumptions you made about the change.** At the end of every push reply — after the PR link — list the three most consequential assumptions you made while implementing the change (about requirements, scope, data, naming, or behavior). Order them by how much the change would have to change if the assumption is wrong, most impactful first. Keep each to one line, and phrase them so the user can quickly confirm or correct. If you genuinely made fewer than three material assumptions, list what you have.

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
