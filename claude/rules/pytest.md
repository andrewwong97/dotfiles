# Running Pytest

All backend tests live in `backend/<APP>/tests/`. To run them:

```bash
cd backend && source venv/bin/activate && python -m pytest <APP>/tests/ -x -q
```

Key points:
- You **must** `cd backend` first — running from the repo root will fail with "file or directory not found".
- Activate the virtualenv with `source venv/bin/activate` (do not use `python3` or system Python).
- Tests are under `<APP>/tests/`, **not** a top-level `tests/` directory.
- Use `-x` to stop on first failure, `-v` for verbose output.
- To run a single test file: `python -m pytest <APP>/tests/<test_file>.py -x -v`
- Mock DB queries that use `.options(joinedload(...))` need the `.options()` step in the mock chain: `mock_db.query.return_value.options.return_value.filter.return_value.first.return_value = ...`
- If `backend/venv` doesn't exist, create it and install dependencies:
  ```bash
  cd backend && python3 -m venv venv && source venv/bin/activate && pip install -r requirements.txt
  ```
