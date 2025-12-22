# CLAUDE.md

## Build & Development Commands

```bash
# Setup development environment (creates .venv, installs pre-commit hooks)
task dev

# Format code
task format

# Lint code
task lint
# or directly: uvx ruff check

# Run full test matrix (Python 3.10-3.12 × dbt 1.8-1.9)
task test

# Quick single-version test
uv run pytest

# Run single test
uv run pytest tests/test_file.py::test_name -v

# Generate dbt manifest before running tests (required)
uv run dbt parse --project-dir demo_duckdb --profiles-dir demo_duckdb -t test
```

## Architecture

```
src/dbt_osmosis/
├── cli/           # Click-based CLI
│   └── main.py    # Commands: yaml, run, workbench, diff
├── core/          # Business logic
│   ├── config.py      # dbt project configuration & manifest parsing
│   ├── transforms.py  # Transform pipeline (>> operator for chaining)
│   ├── llm.py         # Multi-provider LLM integration
│   ├── context.py     # Schema/column context builders
│   └── utils.py       # YAML utilities
└── workbench/     # Streamlit-based SQL IDE
    ├── app.py         # Main application
    └── components/    # Dashboard widgets
```

**Entry point**: `dbt_osmosis.cli.main:cli`

## Key Patterns

- **Transform Pipeline**: Operations chained with `>>` operator in `transforms.py`
- **YAML Handling**: ruamel.yaml preserves comments and formatting
- **Plugin System**: pluggy for extensibility

## Test Setup

Tests require a parsed dbt manifest from the demo_duckdb project. Run `uv run dbt parse --project-dir demo_duckdb --profiles-dir demo_duckdb -t test` before pytest if the manifest is missing.
