# Git Workflow Rules

## Ticket Pickup

**Before starting work on a ticket, claim it by moving its status forward so no one else picks it up.** Do this the moment you begin — before (or alongside) creating the branch/worktree, and before writing any code. Advance the status through the full chain toward "in progress" in order (e.g. `Backlog`/`To Do` → `Planned`/`Selected` → `In Progress`), matching whatever intermediate states the project's workflow defines.

- If the ticket is already partway along (e.g. already `Planned`), run only the remaining transitions needed to reach in-progress. Never move a ticket backwards.
- Transition ids are often source-state dependent — the same target status can have a different id depending on the current status. If a transition fails or the ticket isn't in the status you expect, query the available transitions for the ticket's *current* status and transition by the returned id.
- Also assign the ticket to yourself if it isn't already.

(Project-specific status names, transition ids, and assignment conventions belong in project memory/skills, not here.)

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
6. **Squash unpushed commits** (see **Local Squash**). Before pushing, consolidate the local commits that have **not yet been pushed** into a clean, logically-grouped set — typically one commit per logical change, fix-ups folded into the commit they fix. Only ever reshape *unpushed* commits; a commit already on the remote is frozen.
7. Push with `-u` to set upstream: `git push -u origin <branch>`. See **Push Cadence** — push promptly on the *first* push, but hold off on subsequent pushes until you've consulted the user.
8. **After every push, surface the PR link in your reply** (see **PR Creation**) — even when no other summary is warranted.

## Push Cadence

The **first** push on a new branch should happen promptly: it triggers CI/CD and opens the PR, surfacing pipeline feedback early. Don't sit on the initial push.

**After that, do not eagerly push every commit.** Let commits accumulate locally and **consult the user before each subsequent push.** This leaves a window to consolidate similar or fix-up commits — e.g. three iterations on the same file that should be one commit — before they reach the remote.

The AI **squashes the unpushed commits itself** before pushing (see **Local Squash**) — this is safe because nothing has been shared yet, so it needs no force-push. The consolidation must happen **before** the next push: once commits are pushed, squashing them would require forbidden history rewriting. So the cadence is: let commits accumulate locally, consult the user before each subsequent push, then squash the still-unpushed commits into a clean set as part of that push.

## Local Squash

Before a push, collapse the **unpushed** local commits into a clean, logically-grouped set so the remote history reads as intentional changes, not a trail of iterations and fix-ups.

**The boundary is absolute: only commits that have never been pushed may be reshaped.** A commit that exists on the remote (reachable from the upstream tracking branch) is frozen forever — see Prohibited Operations.

1. **Find the unpushed range.** `git log --oneline @{u}..HEAD` (or `origin/<branch>..HEAD` if no upstream is set yet). Everything listed is unpushed and fair game; everything below `@{u}` is already pushed and off-limits.
2. **Decide the target shape.** Usually one commit for the whole branch on the first push; on later pushes, fold each new fix-up into the new commit it belongs with. Keep genuinely distinct logical changes as separate commits.
3. **Squash with a soft reset (no rebase — interactive rebase isn't available here anyway).** Reset to the boundary, then re-commit:
   ```bash
   git reset --soft @{u}        # first push with no upstream: use the merge-base, e.g. $(git merge-base HEAD origin/<primary>)
   git commit -m "<clear message>"   # or several commits via staged groups, if keeping logical splits
   ```
   This moves only the branch pointer + index; the working tree is untouched, and no pushed commit changes SHA.
4. **Never squash across a base-branch merge.** If the unpushed range includes a `git merge origin/<primary>` (a sync — see Merge Conflicts and Branch Sync), leave that merge commit intact; squash only the work commits around it, or just push without squashing. Don't flatten an integration merge into your own commit.
5. **Re-run the pre-push gate** (tests / prek / code-review) after squashing, since the staged result is a new commit.

If the only commits that *should* be combined are already pushed, do **not** squash them — that needs a force-push (forbidden). Leave them as-is or ask the user.

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

**State the top 3 assumptions you made about the change.** List the three most consequential assumptions you made while implementing the change (about requirements, scope, data, naming, or behavior) in **both** places:
- **In the PR description** — as a final `## Assumptions` section, so reviewers see them alongside the diff.
- **In the push reply** — after the PR link.

Order them by how much the change would have to change if the assumption is wrong, most impactful first. Keep each to one line, and phrase them so the reader can quickly confirm or correct. If you genuinely made fewer than three material assumptions, list what you have.

## Merge Conflicts and Branch Sync

**If the PR has merge conflicts, resolve them by merging the primary branch in — never by rebasing** (see Prohibited Operations). Detect conflicts with:

```bash
gh pr view <branch> --json mergeable,mergeStateStatus
# mergeable: CONFLICTING (or mergeStateStatus: DIRTY) → sync needed
```

To sync:

1. `git fetch origin && git merge origin/<primary>` (the PR's base branch — e.g. `develop` or `main`).
2. Resolve the conflicts. Resolve conservatively — keep both sides' intent; if a conflict involves a substantive logic decision you can't make confidently, stop and ask the user instead of guessing.
3. Re-run the related tests and `prek` (the merge result is new code, even if your own diff didn't change).
4. Commit the merge and push.

**Sync pushes are exempt from the consult-before-pushing cadence** — they contain no new work, only integration, and an unpushed conflict resolution blocks review. Push them promptly.

**Keep syncing while the PR is open.** Whenever the base branch moves and the PR becomes conflicting again, repeat the steps above. The merge-monitoring loop (below) is the natural place to catch this: each poll should also check `mergeable` and run the sync when it reports `CONFLICTING`.

## PR Merge Monitoring and Ticket Closure

**After creating a new PR** (first push on a branch), start a loop to monitor for the merge and close any linked Jira ticket:

1. **Identify the ticket key.** Check in order:
   - Branch name: look for the project's ticket-key pattern (e.g. `bugfix/abc-1758-fix-projections` → `ABC-1758`)
   - PR title: look for the same pattern
   - If no ticket is found, still monitor for the merge but skip ticket closure.

2. **Invoke `/loop`** with a prompt that polls `gh pr view <branch> --json state,mergeable -q '{state: .state, mergeable: .mergeable}'` and acts on the result:
   - If not `MERGED` and `mergeable` is `CONFLICTING`: run the sync procedure from **Merge Conflicts and Branch Sync** (merge the base branch in, resolve, test, push), then continue looping.
   - If not `MERGED`: continue looping (5-minute interval is appropriate — human review takes time).
   - If `MERGED`:
     - If a ticket was identified, close it by transitioning it to the project's done/closed status. Query the ticket's available transitions to find the right one rather than assuming an id. If the done transition has no field screen, set the resolution in a separate edit call afterwards.
     - Report the merge (and ticket closure if applicable) to the user.
     - Stop the loop (omit the `ScheduleWakeup` call).

3. **Do not start a second loop** on subsequent pushes to the same branch — one loop per PR is enough.

(Project-specific ticket-key patterns, transition ids, and tracker coordinates — e.g. Jira cloudId — belong in project memory/skills, not here.)

## Prohibited and Restricted Operations

### Strictly Forbidden — No Exceptions

**The invariant: a commit that has been pushed to a remote is frozen forever. Its SHA must never change. Force-pushing is never allowed.** AI agents must **never** rewrite *pushed* history, and explicit user approval does not override this. Forbidden, no exceptions:

- `git push --force` or `git push --force-with-lease` — never, under any circumstances
- `git rebase` (interactive or otherwise) that includes any **already-pushed** commit
- `git commit --amend` on a commit that has already been pushed
- `git reset` that removes, undoes, or reshapes any **already-pushed** commit
- Rewriting commits on a shared branch (`master`/`main`/`develop`, or any branch others may have pulled)
- Any other operation that changes the SHA of a commit that exists on the remote

If a situation seems to call for rewriting **pushed** history, stop, explain, and ask the user to perform it manually.

### Allowed — Reshaping Unpushed Commits

Consolidating commits that have **never been pushed** is explicitly allowed and is part of the normal flow (see **Local Squash**): `git reset --soft` to the upstream/merge-base then re-commit, or `git commit --amend` on an unpushed commit. These touch only local, unshared history and need no force-push. The test is simple — `git log @{u}..HEAD` lists what is reshapeable; anything below `@{u}` is not.

### Requires Explicit User Approval

- Deleting a branch
- Merging or closing a PR
