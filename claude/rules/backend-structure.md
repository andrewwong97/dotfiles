# Backend Directory Structure Rules

The backend Python package lives in `backend/<APP>/`. Follow these conventions when adding new modules, moving files, or deciding where code belongs.

## Package Layout

```
<APP>/
├── api/             # FastAPI app, routes, Pydantic request/response schemas
│   ├── routes/      # One file per resource (e.g. <resource>_resource.py)
│   └── schema/      # Pydantic models for API input/output
├── cloud/           # External cloud service integrations
│   ├── ai/          # AI/LLM provider implementations
│   ├── storage/     # Object storage implementations
│   └── factory.py   # Provider factory — selects implementation by env config
├── db/              # Database only — ORM models, session factory, seed data, migrations
│   ├── models/      # SQLAlchemy model classes
│   └── seed_data/   # Static seed data for dev/test
├── engine/          # Domain-specific processing pipeline
├── export/          # File generation and output logic
├── jobs/            # Background job queue and handlers
├── services/        # Business logic and cross-cutting concerns (auth, domain ops)
├── tests/           # Test files mirroring the implementation layout
│   ├── api/
│   ├── cloud/
│   ├── db/
│   ├── engine/
│   ├── services/
│   └── workers/
└── workers/         # Long-running consumers
```

## Placement Rules

- **API schemas** (Pydantic `BaseModel` for request/response) go in `api/schema/`, never in `db/`.
- **ORM models** (SQLAlchemy) go in `db/models/`. Do not mix Pydantic API schemas into `db/`.
- **Cloud clients** go in `cloud/`. Storage clients go in `cloud/storage/`, AI clients go in `cloud/ai/`. Do not put cloud/storage code in `db/`.
- **Business logic** that doesn't belong to a route handler goes in `services/`. Auth, token management, and domain services live here.
- **AI/ML pipeline code** (preprocessing, inference, classification) goes in `engine/`.
- **Top-level orphan files** are not allowed in the `<APP>/` package root. Every module belongs in a subpackage.

## Naming

- Use **snake_case** for all new Python file names (e.g. `text_only_pipeline.py`, not `TextOnlyPipeline.py`).
- Route files: `<resource>_resource.py` (e.g. `user_resource.py`).
- Schema files: `<domain>_schema.py` (e.g. `user_schema.py`).
- Test files: `test_<module>.py`, placed in the matching `tests/<package>/` subdirectory.

## Anti-Patterns

- **No stub files** for unimplemented providers. Don't add placeholder modules with only `TODO` or `pass`. Add them when there's real code.
- **No duplicate abstractions** for the same service. There should be one module per cloud provider — not two packages covering the same service.
- **No vague package names** like `impl/`, `utils/`, or `helpers/`. Name packages after what they contain.
- **No flat test dumps.** Tests must be organized into subdirectories that mirror the implementation packages.

## Adding a New Domain

When adding a genuinely new domain (e.g. `notifications/`, `billing/`):

1. Create the package under `<APP>/`.
2. Create a matching `tests/<package>/` directory with an `__init__.py`.
3. If it needs API routes, add a `<domain>_resource.py` in `api/routes/` and schemas in `api/schema/`.
4. If it needs a cloud provider, add the implementation in the appropriate `cloud/` subpackage and register it in `cloud/factory.py`.
