Run readonly verification queries against the dev postgres database to confirm schema and data integrity.

## Rules — Read This First

**Every query must be approved by the user before running.** Present each query with full visibility of what it does before executing. Never batch queries silently.

**Only the following operations are permitted:**
- `SELECT` statements
- `\d`, `\di`, `\dt`, `\dn` psql meta-commands (interactive session only — use `information_schema` queries when running via `-c`)
- `information_schema` and `pg_catalog` queries
- `pg_indexes` and `pg_catalog` system table reads

**Never run** `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`, or any other write operation directly against the database. The database is a shared dev resource. If a fix is needed, it goes in a migration file — never applied manually.

---

## How to Run a Query

> **Note for the user:** You may be prompted **twice** per query — once by the agent's `AskUserQuestion` (showing you the SQL to review) and once by Claude Code's permission system when the `Bash` tool executes the `docker exec` command. Both prompts are for the same query. This is expected behavior when the Bash tool requires explicit approval in your Claude Code permission profile.

For every query, use `AskUserQuestion` to present it **before** executing. Show the SQL in the question body so the user can read it clearly before choosing. Use this structure:

```
Question: "Run this readonly query to verify <what you're checking>?\n\n```sql\n<SQL here>\n```\n\nTarget: <DB_NAME> @ <PROJECT>-postgres-1\nAccess: read-only SELECT — no data modified"
Options:
  - "Yes, run it"  (description: brief note on what it checks)
  - "Skip"         (description: skip this step)
```

Only proceed if the user selects "Yes, run it". Then execute via:

```bash
docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c "<SQL>"
```

> **Note:** Backslash meta-commands like `\d` work reliably when passed via `-c` in this project's postgres container. If a meta-command produces no output, fall back to the equivalent `information_schema` query shown below.

---

## Common Verification Queries

All queries below require user approval via `AskUserQuestion` before running. Containers must be running (`make up` or `docker-compose up -d`) before executing any `docker exec` command.

### Table structure
```bash
docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c '\d <table_name>'
```
Fallback (if `\d` produces no output):
```sql
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = '<table_name>' AND table_schema = 'public'
ORDER BY ordinal_position;
```

### All indexes on a table
```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = '<table_name>';
```

### Foreign key constraints
```sql
SELECT tc.constraint_name, kcu.column_name,
       ccu.table_name AS foreign_table, ccu.column_name AS foreign_column,
       rc.delete_rule
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name AND kcu.table_name = tc.table_name
JOIN information_schema.constraint_column_usage ccu
  ON tc.constraint_name = ccu.constraint_name
JOIN information_schema.referential_constraints rc
  ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND tc.table_name = '<table_name>'
  AND tc.table_schema = 'public';
```

### Row count
```sql
SELECT COUNT(*) FROM <table_name>;
```

### Recent rows (for audit/token tables)
```sql
SELECT * FROM <table_name> ORDER BY created_at DESC LIMIT 10;
```

### Check a specific column exists with expected type
```sql
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = '<table_name>'
  AND column_name = '<column_name>'
  AND table_schema = 'public';
```

### List all tables in the database
```sql
SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;
```

### Check alembic migration history (also requires approval)
```bash
docker exec <PROJECT>-api-1 alembic history --verbose
```

### Current alembic head (also requires approval)
```bash
docker exec <PROJECT>-api-1 alembic current
```

---

## Interpreting Results

- **Missing column or table**: the migration's `CREATE TABLE IF NOT EXISTS` or `ADD COLUMN IF NOT EXISTS` may have silently skipped due to a pre-existing conflicting object. Use the `information_schema.columns` query to compare actual vs expected schema.
- **Missing index**: the `CREATE INDEX IF NOT EXISTS` may have skipped. Check `pg_indexes` for any index on that column under a different name.
- **Missing FK**: check `information_schema.referential_constraints` directly for the constraint name.
- **Unexpected rows**: investigate whether a previous test or seed run left data. Do not delete — flag for human review.
- **`NOTICE:`, `HINT:`, `DETAIL:` log lines**: these are informational and expected. Postgres emits `NOTICE` on `IF NOT EXISTS` no-ops — do not flag these as errors.
