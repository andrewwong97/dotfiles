# ORM Schema Change Rules

When adding columns, relationships, or foreign keys to SQLAlchemy models, follow this checklist to avoid cascading serialization failures.

## 1. Audit Every Query That Touches the Model

If you add a relationship (e.g. `Item.file`) or a field that Pydantic serializes via `from_attributes = True`:

- **Search for all `db.query(Model)` calls** across the codebase.
- Any endpoint whose `response_model` includes the new field **must** eagerly load it with `joinedload()`.
- Lazy loading will silently trigger extra queries — or crash if the session is closing.

```python
# BAD — Pydantic reads order.customer, triggers lazy load
db.query(Order).options(joinedload(Order.category)).all()

# GOOD — explicitly load all relationships Pydantic will access
db.query(Order).options(joinedload(Order.category), joinedload(Order.customer)).all()
```

## 2. Keep Pydantic Schema and ORM Loads in Sync

When you add an `Optional[RelatedSchema]` field to a Pydantic schema with `from_attributes = True`, that field **will be read** during serialization. The ORM query must provide it.

Checklist for adding a relationship field to a schema:
1. Add the column + relationship to the SQLAlchemy model.
2. Add the optional field to the Pydantic schema.
3. **Grep for every endpoint** that returns this schema and add `joinedload()`.
4. Verify the frontend type marks the field as optional (`field?: Type`).

## 3. Legacy Data Compatibility

New nullable columns (`nullable=True`) will be `NULL` for all existing rows. Ensure:
- The Pydantic field has a default (`= None`).
- The frontend type marks it optional (`?`).
- Any code that branches on the new field (e.g. `if item.file_id:`) has a working fallback for `NULL`.
- Accessor methods gracefully handle both old and new paths.

## 4. Backwards-Compatible Schema Changes

Migrations and code deploys do **not** always happen at the same time. Code must remain robust when running against either the old or new schema during the transition window. This means every database change should be planned as at least two separate steps:

### Adding a Column

1. **First PR — migration only**: Add the column (nullable, with a safe default). Deploy and run the migration. Existing code keeps working because it never references the new column.
2. **Second PR — code change**: Add the column to the ORM model, Pydantic schema, and queries. By the time this code runs, the column already exists in the DB.

If both must ship in the same PR, the ORM model column **must** be nullable with `default=None` and the code must handle `None` gracefully — but prefer the two-step approach.

### Removing a Column

1. **First PR — code change**: Remove all references to the column from the ORM model, Pydantic schemas, and queries. Deploy. The column still exists in the DB but nothing reads or writes it.
2. **Second PR — migration only**: Drop the column. Safe because no running code references it.

### Renaming a Column

Treat as an add + remove:
1. Add the new column (migration).
2. Dual-write to both old and new columns; read from new with fallback to old.
3. Backfill old rows.
4. Remove old-column references from code.
5. Drop old column (migration).

### Why This Matters

If the ORM model references a column that doesn't exist yet, SQLAlchemy includes it in every `SELECT` and **all queries on that table will fail**. Conversely, if a migration drops a column that running code still reads, the same crash occurs. The two-step approach eliminates both failure modes.

### Pragmatism Over Ceremony

Prefer the simplest safe approach. Adding a nullable column and removing an unused column are cheap two-step operations — always do those correctly. But multi-step dual-write migrations (like renames or type changes) are expensive and slow development. **Bias against designs that require dual writes.** Look for alternatives first: can you add a new column instead of renaming? Can you keep the old type and transform at the application layer?

That said, if the correct solution genuinely requires a dual-write migration, do it properly — don't cut corners on data integrity to save time. The goal is to avoid *unnecessary* complexity, not to avoid *necessary* correctness.

### Confidence Check

If you are **70% or less confident** that a schema change is backwards-compatible — or that the chosen migration strategy is safe — **stop and raise the concern to the user** before proceeding. Explain what you're unsure about and what the risks are. Do not guess your way through database changes.

### Rule of Thumb

> Every migration must be safe to run **before or after** the corresponding code deploy. If a migration would break currently-running code, split it into smaller steps until each step is independently safe.

## 5. Test Both Paths

When adding a new optional relationship, write tests for:
- Items **with** the relationship populated (new data).
- Items **without** it (legacy data, field is `NULL`).
- Mixed listings (both types in the same query result).
