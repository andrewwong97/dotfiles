# Feature Flags

The app uses a **DB-backed feature flag system** scoped per environment (`development`, `test`, `production`). Flags are stored in the `feature_flags` table and seeded on app startup.

## Architecture

| Layer | File | Purpose |
|-------|------|---------|
| Model | `backend/<APP>/db/models/feature_flag.py` | SQLAlchemy model — `FeatureFlag` |
| Seed registry | `backend/<APP>/db/models/seed_data/feature_flags.py` | `FEATURE_FLAG_SEED_DATA` list defining all flags and per-env defaults |
| Service | `backend/<APP>/services/feature_flags.py` | `is_flag_enabled(db, flag_name)` and `get_flag(db, flag_name)` helpers |
| Seeder | `backend/<APP>/api/app.py` → `seed_feature_flags()` | Runs on startup; upserts all flags from seed data (inserts missing, updates enabled value for existing) |
| Tests | `backend/<APP>/tests/services/test_feature_flags.py` | Unit tests for model, seed data, and service helpers |

## Table Schema

```
feature_flags
  id          SERIAL PRIMARY KEY
  flag_name   VARCHAR NOT NULL        -- e.g. "ENABLE_AI"
  environment VARCHAR NOT NULL        -- "development", "test", or "production"
  enabled     BOOLEAN NOT NULL DEFAULT false
  metadata    JSON                    -- optional: description, owner, etc.
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
  UNIQUE (flag_name, environment)
```

## Checking a Flag in Code

```python
from <APP>.services.feature_flags import is_flag_enabled

# Reads ENVIRONMENT env var automatically:
if is_flag_enabled(db, "ENABLE_FEATURE_Y"):
    ...

# Or specify environment explicitly:
if is_flag_enabled(db, "ENABLE_FEATURE_Y", environment="production"):
    ...
```

`is_flag_enabled` is **fail-closed** — returns `False` if the flag is not found in the DB.

## Adding a New Flag

1. Add an entry to `FEATURE_FLAG_SEED_DATA` in `backend/<APP>/db/models/seed_data/feature_flags.py`:
   ```python
   {
       "flag_name": "MY_NEW_FLAG",
       "defaults": {
           "development": True,
           "test": False,
           "production": False,
       },
       "metadata": {
           "description": "What this flag controls",
           "owner": "team-name",
       },
   },
   ```
2. Restart the app (or redeploy). The seeder will insert the new `(flag_name, env)` rows automatically.
3. Use `is_flag_enabled(db, "MY_NEW_FLAG")` in your code.

No migration is needed to add new flags — only to change the table schema itself.

## Toggling a Flag

Change the `defaults` value in `FEATURE_FLAG_SEED_DATA` and redeploy. The seeder upserts on startup — it will update the `enabled` value and `updated_at` timestamp for any flag whose DB value differs from the seed data.

**Note:** Direct DB edits to `enabled` will be overwritten on the next restart by the seed data defaults. Seed data is the source of truth.

## Current Flags

| Flag | Dev | Test | Prod | Description |
|------|-----|------|------|-------------|
| `ENABLE_FEATURE_X` | on | on | on | Example: background processing pipeline |
| `ALLOW_FEATURE_Z` | off | off | off | Example: gated access control |
| `ENABLE_FEATURE_Y` | off | off | on | Example: optional processing capability |
