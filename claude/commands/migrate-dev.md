Run `make migrate-dev` to apply pending Alembic migrations against the dev database and verify the result.

## Steps

1. Run the migration and capture all output:
   ```bash
   make migrate-dev
   ```

2. **Check step [1/4] — current head**: note the revision hash (or "(head)" / "No current heads" if the DB is fresh).

3. **Check step [2/4] — upgrade output**: confirm alembic printed each migration that ran (e.g. `Running upgrade abc123 -> def456, ...`). If the command failed, note the exact error message and the migration file it came from.

4. **Check step [3/4] — new head**: verify it now shows `(head)` next to the latest revision hash. If it does not match, the upgrade did not complete.

5. **Check step [4/4] — postgres logs**: scan for any `ERROR`, `FATAL`, or `PANIC` lines. Each one represents a problem that needs investigation. `NOTICE:`, `HINT:`, and `DETAIL:` lines are informational and expected — Postgres emits `NOTICE` on `IF NOT EXISTS` no-ops, so do not flag these.

6. **Verify schema against migration files** — for each migration that ran in step [2/4], read the corresponding file in `backend/migrations/versions/` and cross-check every DDL statement against the live database. Containers are already running from step 1. Use **read-only** introspection only — see Hard Rules below. See also the `/db-verification` skill for query templates.

   For each table created or modified by the migration, run:
   ```bash
   # List columns and types
   docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c '\d <table_name>'

   # Verify indexes exist
   docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c \
     "SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table_name>';"

   # Verify foreign keys
   docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c "
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
       AND tc.table_schema = 'public';"

   # Verify row counts (for seed data migrations)
   docker exec <PROJECT>-postgres-1 psql -U <DB_USER> -d <DB_NAME> -c "SELECT COUNT(*) FROM <table_name>;"
   ```

   Check each result against what the migration's `upgrade()` function specifies:
   - Every `ADD COLUMN IF NOT EXISTS` → column appears with correct type and nullability
   - Every `CREATE TABLE IF NOT EXISTS` → table exists with all expected columns
   - Every `CREATE INDEX IF NOT EXISTS` → index appears in `pg_indexes`
   - Every `REFERENCES` FK → foreign key constraint appears with correct `delete_rule`

7. **Full postgres log scan** — review the complete recent log for anything unexpected:
   ```bash
   docker logs <PROJECT>-postgres-1 --tail=200 2>&1
   ```
   Normal entries are `LOG:`, `NOTICE:`, `HINT:`, and `DETAIL:` lines. Flag only `WARNING:`, `ERROR:`, `FATAL:`, or `PANIC:` lines.

## Investigating Errors

For each error found in steps 2–7, investigate one at a time:

- **Alembic upgrade failure**: open the migration file in `backend/migrations/versions/` that was running when the error occurred. Read the failing `op.execute()` call and the postgres error message together to determine the root cause (e.g. column already exists, type mismatch, FK violation).
- **Postgres log error**: correlate the timestamp with the migration that was running. Check whether the SQL in the migration file matches what postgres rejected.
- **Head mismatch after upgrade**: run `docker exec <PROJECT>-api-1 alembic history --verbose` to see the full chain and identify any gap or branch.
- **Schema mismatch**: if a column, index, or constraint is missing from the live DB despite the migration appearing to succeed, check whether the `IF NOT EXISTS` guard silently skipped the statement due to a pre-existing object with a conflicting definition.

Fix the issue in the migration file following the conventions in the alembic-migrations rule, then re-run `make migrate-dev` to confirm it passes cleanly.

## Hard Rules

- **Never run write commands directly against the database** during verification (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`). All schema changes go through `make migrate-dev` and the migration files — never applied manually.
- All verification in steps 6–7 uses `SELECT`, `\d`, `pg_indexes`, or `information_schema` queries only.
- If a fix is needed, it goes in the migration file — never applied manually to the database.
