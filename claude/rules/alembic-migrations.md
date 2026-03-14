---
description: Alembic database migration conventions and safety guardrails
paths: ["backend/migrations/**/*.py"]
---

# Alembic Migration Rules

Migrations live in `backend/migrations/versions/`. When generating migrations, always use the Makefile target:

```bash
make create-migration m="short_descriptive_message"
```

It is very important to do so, because a running API container with alembic installed will tell you the migration history and it will also autogenerate the next hash.

This spins up dev Docker containers (if not already running), runs `alembic revision` inside the API container, and places the blank file in `backend/migrations/versions/`. If the containers aren't up, the target starts and stops them automatically.

After generating, fill in the `upgrade()` and `downgrade()` functions with your SQL. The revision ID and chain are set by Alembic — never edit those by hand.

**After creating a new migration, update `backend/migrations/HEAD`** to contain the new revision ID. This single-line file prevents two PRs from merging conflicting migration chains — git will flag a merge conflict if both PRs change it. CI tests (`test_migration_chain.py`) verify the HEAD file matches the actual chain head.

## Writing Upgrade / Downgrade

Use **direct SQL via `op.execute()`**, not `op.add_column()` or `op.create_table()` helpers.

Always add idempotency guardrails:

```python
# ✅ GOOD — safe to re-run
op.execute("ALTER TABLE items ADD COLUMN IF NOT EXISTS new_col BOOLEAN DEFAULT false")
op.execute("CREATE TABLE IF NOT EXISTS audit_logs ( ... )")
op.execute("CREATE INDEX IF NOT EXISTS ix_audit_logs_item_id ON audit_logs(item_id)")

# ✅ GOOD — safe downgrade
op.execute("ALTER TABLE items DROP COLUMN IF EXISTS new_col")
op.execute("DROP INDEX IF EXISTS ix_audit_logs_item_id")
op.execute("DROP TABLE IF EXISTS audit_logs")

# ❌ BAD — crashes on partial re-run
op.add_column('items', sa.Column('new_col', sa.Boolean))
op.create_table('audit_logs', ...)
```

## Downgrade Order

Drop dependents before parents: indexes → columns/FKs → tables. Mirror the upgrade in reverse.

## Docstrings

Include a module-level docstring explaining **what** the migration does and **why**, plus inline comments grouping logical steps inside `upgrade()` and `downgrade()`.

## Online Migration Strategy (Stripe 4-Phase Pattern)

All schema changes on live tables must be planned as **online (zero-downtime) migrations**. Each phase is a separate migration file and a separate deploy. Every migration must be backwards-compatible with the **previous** application version — old code must continue to work while the deploy is in flight.

### The 4 Phases

**Phase 1 — Expand**: Add the new structure without removing anything. New columns must be nullable with no application-level default (or use a safe `DEFAULT` that Postgres can apply without a table rewrite). Use `CREATE INDEX CONCURRENTLY` for new indexes.

**Phase 2 — Backfill**: Populate existing rows with correct values, in batches to avoid long-running transactions. No application code changes in this phase.

**Phase 3 — Deploy new code**: Update application code to read/write the new column. Old structure still present; old code path still works.

**Phase 4 — Contract**: After all instances run new code, clean up: add `NOT NULL`, drop old columns/indexes, remove dual-write logic.

### Specific Rules

**Never add a `NOT NULL` column without a default in one step** — Postgres rewrites the whole table, causing a full table lock. Always: add nullable → backfill → add constraint.

**Always use `CONCURRENTLY` for indexes** — avoids table lock on live tables:
```python
# ✅ GOOD
op.execute("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_items_user_id ON items(user_id)")

# ❌ BAD — locks the table
op.execute("CREATE INDEX IF NOT EXISTS ix_items_user_id ON items(user_id)")
```

**Use `NOT VALID` + `VALIDATE CONSTRAINT` for new foreign keys** — splits the lock into two cheap operations:
```python
# Phase 1: add constraint without scanning existing rows (fast, low lock)
op.execute("""
    ALTER TABLE items
    ADD CONSTRAINT fk_items_user_id FOREIGN KEY (user_id) REFERENCES users(id)
    NOT VALID
""")

# Phase 4: validate existing rows (no lock on writes, just a share lock)
op.execute("ALTER TABLE items VALIDATE CONSTRAINT fk_items_user_id")
```

**One logical change per migration file** — keep migrations small and independently reversible.

## Reference

See recent migrations in `backend/migrations/versions/` for good examples of these patterns.
