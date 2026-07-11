# CLAUDE.md

These are instructions which I use to instruct Claude on how to operate using .claude/ config.

# Claude Code Instructions

## Package Manager & Node

Always use `pnpm` for package and script commands. Run `nvm use` before Node-based tasks to ensure the correct Node version is active.

## Running Frontend Tests

From the repo root, you must first change into the frontend directory and activate the correct Node version:

```bash
cd frontend && nvm use
```

Then run whatever tests are needed, e.g.:

```bash
pnpm test -- --run src/path/to/file.test.tsx
```

## Shell Environment

When the OS is Windows or WSL, run project commands in WSL `zsh`. Do not use PowerShell, `cmd.exe`, or Git Bash. Use Linux-style paths (`/home/...`); use `/mnt/c/...` when referencing Windows filesystem locations from WSL.

## Security & Trust

This product handles sensitive user financial documents (bank statements, net worth docs). User trust and security are top priorities in every feature decision:
- Treat all uploaded materials as highly sensitive.
- Minimize sensitive data in logs, debug output, analytics, and error messages.
- Avoid unsafe defaults; require explicit user intent for destructive or externally visible actions.

## Product Vision

<short blurb of what the project does>

### Core principles
<a bulleted summary of core principles to always consider when making a decision>

## Skills
- `frontend-design`: Use for component/page/app UI implementation and redesign tasks. Auto-invoked when user requests frontend UI work, visual polish, or component/page creation.
- `migrate-dev`: Run `/migrate-dev` to apply pending Alembic migrations against the dev DB, verify the new head, and investigate any errors one by one.

## Scoped Rules

- @.claude/rules/alembic-migrations.md
- @.claude/rules/orm-schema-changes.md
- @.claude/rules/storybook.md
- @.claude/rules/rtk-query-api.md
- @.claude/rules/frontend-testing.md
- @.claude/rules/git-workflow.md
- @.claude/rules/feature-flags.md
- @.claude/rules/backend-structure.md
# Agent Instructions

## Non-Interactive Shell Commands

**ALWAYS use non-interactive flags** with file operations to avoid hanging on confirmation prompts.

Shell commands like `cp`, `mv`, and `rm` may be aliased to include `-i` (interactive) mode on some systems, causing the agent to hang indefinitely waiting for y/n input.

**Use these forms instead:**
```bash
# Force overwrite without prompting
cp -f source dest           # NOT: cp source dest
mv -f source dest           # NOT: mv source dest
rm -f file                  # NOT: rm file

# For recursive operations
rm -rf directory            # NOT: rm -r directory
cp -rf source dest          # NOT: cp -r source dest
```

**Other commands that may prompt:**
- `scp` - use `-o BatchMode=yes` for non-interactive
- `ssh` - use `-o BatchMode=yes` to fail instead of prompting
- `apt-get` - use `-y` flag
- `brew` - use `HOMEBREW_NO_AUTO_UPDATE=1` env var

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
