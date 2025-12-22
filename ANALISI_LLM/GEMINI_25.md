## `dbt-osmosis`: Project Documentation

### 1. Project Overview

`dbt-osmosis` is a Python-based command-line tool and web UI designed to streamline and enhance the dbt development workflow. Its primary goal is to reduce the manual effort involved in creating and maintaining dbt documentation, organizing project files, and iterating on model development.

It consists of two main user-facing components:
1.  **A Command-Line Interface (CLI):** For automating YAML management and launching the web UI.
2.  **A Web-Based Workbench:** A Streamlit application that provides an interactive environment for editing, compiling, and testing dbt models in real-time.

### 2. Conceptual Architecture

The project operates as a layer on top of a standard dbt project. It can be conceptualized with the following flow:

```
+-----------------+      +------------------------+      +----------------------+
|   User / Dev    |----->|   dbt-osmosis Tools    |----->|   User's dbt Project |
+-----------------+      |  (CLI or Workbench)    |      +----------------------+
                         +-----------+------------+                |
                                     |                             | (via dbt-core)
                                     |                             |
                         +-----------v------------+      +---------v------------+
                         | dbt-osmosis Core Engine|----->|   Data Warehouse     |
                         +------------------------+      +----------------------+
```

1.  **User Interaction:** A developer interacts with either the `dbt-osmosis` CLI or the Workbench UI.
2.  **Tools Layer:**
    *   The **CLI** (`cli/main.py`) parses user commands (e.g., `yaml refactor`, `workbench`).
    *   The **Workbench** (`workbench/app.py`) provides the interactive Streamlit interface.
3.  **Core Engine:** Both tools utilize the `core` module, which contains the business logic for all operations. This engine programmatically interacts with the user's dbt project by:
    *   Parsing the `dbt_project.yml` for configuration.
    *   Reading and writing `.yml` schema files.
    *   Reading `.sql` model files.
    *   Leveraging `dbt-core` libraries to compile SQL and introspect the project's structure (e.g., using the `manifest.json`).
4.  **dbt Project & DWH:** The tool reads the project's files directly and, for certain operations like column introspection or query testing, uses dbt's adapters to connect to the data warehouse.

### 3. Component Breakdown

#### 3.1. CLI (`src/dbt_osmosis/cli`)

*   **File:** `cli/main.py`
*   **Framework:** `click` (inferred from `pyproject.toml`).
*   **Purpose:** Defines the command-line entry points.
*   **Commands:**
    *   `dbt-osmosis yaml ...`: A group of commands for YAML management.
        *   `refactor`: The most comprehensive command. It likely orchestrates restructuring (`restructuring.py`), documentation inheritance (`inheritance.py`), and syncing with the database schema (`sync_operations.py`).
        *   `organize`: Focuses solely on file structure and organization, likely driven by `restructuring.py`.
        *   `document`: Focuses on propagating column descriptions, driven by `inheritance.py`.
    *   `dbt-osmosis workbench`: Starts the Streamlit web application.

#### 3.2. Core Engine (`src/dbt_osmosis/core`)

This is the heart of the application.

*   **Configuration (`config.py`, `settings.py`):** Loads and validates configuration from the user's `dbt_project.yml`, likely under an `osmosis:` key.
*   **dbt Project Introspection (`introspection.py`):** Contains functions that parse dbt's `manifest.json`. This file is critical for understanding the entire dbt project's state: models, sources, exposures, columns, tests, and the DAG (Directed Acyclic Graph).
*   **YAML/Schema Management (`core/schema/`):**
    *   `parser.py`, `reader.py`: Use `ruamel.yaml` to safely read dbt schema files, preserving comments and formatting.
    *   `writer.py`: Writes the modified YAML structures back to disk.
    *   This separation of concerns is excellent, making the YAML handling logic robust.
*   **Core Logic Files:**
    *   `inheritance.py`: Implements the logic for propagating documentation from upstream models to downstream models. It likely traverses the dbt DAG provided by `introspection.py`.
    *   `restructuring.py`: Handles the logic for organizing YAML files based on the rules defined in the project configuration (e.g., one file per model, one file per directory).
    *   `sync_operations.py`: Connects to the data warehouse (via dbt) to fetch the current schema for a model (`get_columns_in_relation`). It then diffs this against the existing YAML to add missing columns or flag removed ones.
    *   `sql_operations.py`: Provides functions to compile and execute SQL, which is primarily used by the Workbench.
*   **Plugin System (`plugins.py`):** The use of `pluggy` suggests a plugin architecture, allowing for extensibility. This is an advanced feature that could allow third-party packages to add new commands or modify existing behavior.

#### 3.3. Workbench (`src/dbt_osmosis/workbench`)

*   **File:** `workbench/app.py`
*   **Framework:** Streamlit.
*   **Purpose:** Provides an interactive, web-based IDE for dbt development.
*   **Components (`workbench/components/`):**
    *   `editor.py`: Implements the side-by-side code editor (using `streamlit-ace`) where a user can write dbt-Jinja-SQL. It calls `sql_operations.py` to compile the code in real-time.
    *   `preview.py` / `renderer.py`: Displays the compiled SQL and query results.
    *   `profiler.py`: Integrates `ydata-profiling` to generate a detailed data profile of a model's query results. This is a powerful feature for data exploration.
    *   `dashboard.py`, `feed.py`: Renders the main UI layout, useful links, and the RSS feed seen in the screenshots.

---

## Maintenance and Modernization Plan

Here is a prioritized breakdown of tasks to update and maintain the `dbt-osmosis` fork.

### Priority 1: Dependency Modernization

*This is the most critical step, as outdated dependencies are a security risk and prevent the use of new features.*

1.  **Upgrade `dbt-core`:** The `pyproject.toml` allows `dbt-core` up to `<1.11`. The dbt ecosystem moves quickly.
    *   **Action:** Incrementally upgrade `dbt-core` and `dbt-duckdb`. Start by targeting the latest patch of each minor version (e.g., 1.8.x, then 1.9.x, etc.) and run the test suite at each step.
    *   **Action:** Run `task test` as defined in `Taskfile.yml`. This is the baseline.
    *   **Challenge:** `dbt-core` has breaking changes in its Python APIs between minor versions (especially the `Manifest` object structure). The `introspection.py` file will likely need significant updates.
2.  **Update Other Dependencies:**
    *   **Action:** Use a tool like `uv` or `pip-tools` to update other dependencies like `streamlit`, `ydata-profiling`, and `ruamel.yaml`.
    *   **Action:** Re-generate the `uv.lock` file after updating dependencies.
3.  **Validate Python Versions:** The project currently supports Python 3.10-3.12, which is great. Ensure the test suite continues to run successfully across all versions after the dependency upgrades.

### Priority 2: CI/CD and Test Suite Enhancement

*A robust test suite is essential for confident maintenance.*

1.  **Validate Existing CI:** The workflows in `.github/workflows/` are the ground truth.
    *   **Action:** Trigger the `lint.yml` and `tests.yml` workflows to ensure they are functioning correctly on the `main` branch.
2.  **Expand Test Matrix:**
    *   **Action:** Modify `Taskfile.yml` and `.github/workflows/tests.yml` to include newer versions of dbt-core as you add support for them.
3.  **Increase Test Coverage:**
    *   **Action:** Install `pytest-cov` and run tests with `uv run pytest --cov=src/dbt_osmosis`.
    *   **Action:** Analyze the coverage report to identify critical, untested logic in the `core` module (especially in `inheritance.py`, `restructuring.py`, and `sync_operations.py`). Write new tests to cover these gaps.

### Priority 3: Codebase Health and Refactoring

*Improve the long-term maintainability of the code.*

1.  **Linting and Formatting:**
    *   **Action:** Run `task format` and `task lint` to ensure the entire codebase conforms to the project's style (Ruff, Black).
2.  **Audit for Deprecated Code:**
    *   **Action:** As you upgrade `dbt-core`, consult its changelog and audit `dbt-osmosis` for usage of deprecated internal functions. Refactor to use officially supported APIs where possible.
3.  **Improve Error Handling:**
    *   **Action:** Review the code for generic `except Exception:` blocks. Replace them with more specific exceptions to provide clearer error messages to users. The `logger.py` file should be used to provide structured and helpful output.

### Priority 4: Documentation

*Good documentation is key for attracting new users and contributors.*

1.  **Update the Docusaurus Site:** The `docs/` directory contains a full documentation website.
    *   **Action:** Navigate to the `docs/` directory, run `npm install`, and then `npm run start` to launch the local development server.
    *   **Action:** Update the documentation to reflect all the changes made, especially regarding new dbt versions supported and any changes to CLI commands.
2.  **Update the README:**
    *   **Action:** Edit the `README.md` to reflect the project's new status as an actively maintained fork. Update version numbers and any outdated links.
3.  **Improve Docstrings:**
    *   **Action:** Add or enhance Python docstrings in the `core` module's public functions to clarify their purpose, arguments, and return values. This is invaluable for future maintenance.
