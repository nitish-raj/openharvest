# AGENTS.md

High-signal guidance for working in this repo. Read this before making changes.

## Package manager

Use `uv` exclusively — never call `pip` directly.

```bash
uv sync --all-extras   # install everything (dev + all optional deps)
uv run <command>       # run anything in the project venv
```

`--all-extras` is **required** for type checking and running pre-commit locally, because `ty` resolves lazy imports of optional deps (pandas, polars, dlt, duckdb, pyarrow, openpyxl, airflow). Without it you get `unresolved-import` errors.

## Developer commands

| Task | Command |
|------|---------|
| Full QA (format → lint → typecheck → test) | `just qa` or `make qa` |
| Format only | `uv run ruff format .` |
| Lint only | `uv run ruff check . --fix` |
| Type check | `uv run --all-extras ty check .` |
| Tests | `uv run pytest` |
| Single test file | `uv run pytest tests/unit/domain/test_models.py` |
| Pre-commit (all files) | `uv run pre-commit run --all-files` |
| Install pre-commit hooks | `uv run pre-commit install` |
| Serve docs | `just docs-serve` |
| Build dist | `uv build` |

`just qa` and `make qa` run the same pipeline: ruff format → ruff lint → ty check → pytest.

## Pre-commit

Pre-commit includes **local hooks** for `ty check` and `pytest` (see `.pre-commit-config.yaml`). These run via `uv run`, so they need the project venv. Always invoke pre-commit as `uv run pre-commit`, not bare `pre-commit`.

## Architecture

- **Entry point**: `datasluice.cli.app:app` (Typer app). Not `datasluice.cli:app`.
- **Version**: lives in `datasluice/_version.py` — do NOT move it into `__init__.py`. It's separate to break a circular import with `transport/user_agent.py`.
- **Adapters auto-register**: importing `datasluice.adapters` triggers side-effect registration of all built-in adapters (CKAN, data.gouv, Socrata, custom) into the module-level `registry`.
- **Adapter pattern**: each adapter subpackage has `adapter.py`, `mapper.py`, `pagination.py`, `errors.py`. Mappers translate portal-native JSON into `datasluice.domain` models.
- **Lazy imports**: `formats/` and `integrations/` import heavy optional deps (pyarrow, openpyxl, pandas, etc.) inside functions, not at module top-level. Keep it that way.

## Style conventions

- **Line length**: 120 (ruff).
- **Ruff selects**: E, W, F, I, B, UP (pyupgrade).
- **PEP 695 type params**: use `def func[T](...)` syntax, not `TypeVar`. The project targets Python 3.12+.
- **Typer commands**: use `Annotated[str, typer.Option(...)]` pattern, not `param: str = typer.Option(...)`. The B008 rule (flake8-bugbear) rejects function calls in argument defaults.
- **No comments** in code unless explicitly requested.
- **Docstrings**: Google style. First line is a summary.

## Docs

Built with **Zensical** (MkDocs Material wrapper). Config is `zensical.toml`, not `mkdocs.yml`. Logo lives at `docs/assets/datasluice.png`. API docs auto-generated via mkdocstrings — the `docs/api.md` file uses `::: datasluice` directive.

## CI

GitHub Actions (`.github/workflows/ci.yml`):
- **type-check job** uses `uv run --all-extras ty check .` — if you add new optional deps, the CI must install them.
- Tests run on Python 3.12, 3.13, and 3.14 matrix.
- Coverage threshold: 50% (`fail_under` in pyproject.toml).
