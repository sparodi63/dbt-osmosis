# How dbt-osmosis Workbench Works - Technical Deep Dive

## Architecture Overview

The dbt-osmosis workbench is a Streamlit-based application that provides an interactive development environment for dbt SQL models. It's designed to be lightweight, fast, and easy to use while providing powerful features for dbt development.

## Core Components

### 1. Main Application (`app.py`)

The `app.py` file is the heart of the workbench. It:
- Initializes the Streamlit application
- Sets up the dashboard layout
- Manages application state
- Handles command-line arguments
- Orchestrates the interaction between components

#### Key Functions:

- **`main()`**: Entry point that initializes the app and renders the UI
- **`sidebar()`**: Renders the left sidebar with model selection, profile management, and query templates
- **`compile()`**: Compiles SQL using dbt context
- **`run_query()`**: Executes SQL queries against the database
- **`run_profile()`**: Generates data profile reports
- **`inject_model()`**: Loads a model into the editor
- **`save_model()`**: Saves changes back to disk
- **`change_target()`**: Switches between dbt targets

### 2. UI Components

The workbench uses a modular component-based architecture:

#### `editor.py` - SQL Editor
- Provides a code editor for writing SQL/Jinja
- Supports multiple tabs (SQL, YAML, etc.)
- Handles real-time compilation
- Manages content loading and saving
- Features:
  - Syntax highlighting
  - Line numbers
  - Resizable panels
  - Compilation status indicators

#### `renderer.py` - Compiled SQL Display
- Shows the compiled SQL output
- Side-by-side comparison with source
- Syntax highlighting for compiled SQL
- Resizable panel

#### `preview.py` - Query Execution
- Executes SQL queries against the database
- Displays results in a tabular format
- Shows execution status (running, success, error)
- Handles error messages gracefully
- Features:
  - Column headers
  - Pagination
  - Result count
  - Execution time

#### `profiler.py` - Data Profiling
- Generates comprehensive data profiles using `ydata-profiling`
- Provides statistical analysis
- Creates visualizations (histograms, correlations, etc.)
- Features:
  - Missing values analysis
  - Data type detection
  - Statistical summaries
  - Correlation matrices
  - Sample data preview

#### `dashboard.py` - Layout Management
- Manages the responsive grid layout
- Handles resizing and repositioning of panels
- Provides drag-and-drop functionality
- Supports saving/loading layouts

#### `feed.py` - RSS Feed
- Displays Hacker News feed
- Shows article titles and publication dates
- Provides links to articles and comments
- Auto-refreshes content

### 3. State Management

The workbench uses Streamlit's `session_state` to maintain application state:

```python
state.app = SimpleNamespace(
    model="SCRATCH",
    dashboard=board,
    editor=Editor(...),
    renderer=Renderer(...),
    preview=Preview(...),
    profiler=Profiler(...),
    feed=RssFeed(...),
    # Additional state variables
    query="",
    compiled_query="",
    query_state="",
    query_result_df=None,
    target_name="",
    query_template="",
    # ...
)
```

Key state variables:
- `query`: Current SQL/Jinja code in the editor
- `compiled_query`: Compiled SQL output
- `query_state`: Execution status ("", "running", "success", "error")
- `query_result_df`: Pandas DataFrame with query results
- `model`: Currently selected model (or "SCRATCH")
- `target_name`: Currently selected dbt target
- `query_template`: Template for query execution

## Workflow Execution

### 1. Application Initialization

When the workbench starts:
1. Parse command-line arguments (`--project-dir`, `--profiles-dir`)
2. Discover project and profiles directories if not specified
3. Load dbt profiles
4. Create dbt project context
5. Initialize Streamlit session state
6. Set up dashboard with components
7. Load default query (demo query for demo projects, or scratch prompt)
8. Parse dbt manifest to get list of models
9. Fetch RSS feed content

### 2. User Interaction Loop

The workbench runs in a continuous loop:
1. **Render UI**: Streamlit re-renders the entire UI on each interaction
2. **Handle Events**: User actions trigger callbacks
3. **Update State**: State is modified based on user actions
4. **Re-render**: UI updates to reflect new state

### 3. Key User Actions

#### Model Selection
1. User selects a model from dropdown
2. `inject_model()` is called
3. Model file is read from disk
4. Content is loaded into editor
5. Editor displays the model SQL

#### Compilation
1. User presses Ctrl+Enter or types
2. `compile()` function is called
3. SQL is compiled using dbt context
4. Compiled SQL is displayed in renderer
5. Errors are shown if compilation fails

#### Query Execution
1. User presses Ctrl+Shift+Enter
2. `run_query()` function is called
3. SQL is executed against the database
4. Results are fetched and stored in DataFrame
5. Results are displayed in preview panel
6. State is updated to "success" or "error"

#### Profiling
1. User clicks "Run Profile" button
2. `run_profile()` function is called
3. `ydata-profiling` generates report
4. HTML report is displayed in profiler panel

#### Saving Models
1. User clicks "Save" button or presses Ctrl+S
2. `save_model()` function is called
3. Content is written back to disk
4. Confirmation is shown

#### Target Switching
1. User selects different target from sidebar
2. `change_target()` function is called
3. dbt context target is updated
4. Manifest is reloaded
5. Current query is recompiled

## Technical Implementation Details

### Dependencies

The workbench requires several key dependencies:

```python
# Core dependencies
import streamlit as st
from streamlit import session_state as state
from streamlit_elements_fluence import elements, event, sync

# Data processing
import pandas as pd
import ydata_profiling

# dbt integration
from dbt_osmosis.core.osmosis import (
    DbtConfiguration,
    _reload_manifest,
    compile_sql_code,
    create_dbt_project_context,
    discover_profiles_dir,
    discover_project_dir,
    execute_sql_code,
)

# RSS feed
import feedparser

# Utilities
import argparse
import decimal
import os
import sys
import typing as t
from collections import OrderedDict
from datetime import date, datetime
from textwrap import dedent
from types import SimpleNamespace
```

### dbt Integration

The workbench integrates with dbt through the `dbt_osmosis.core.osmosis` module:

```python
# Create dbt project context
app.ctx = create_dbt_project_context(
    config=DbtConfiguration(
        project_dir=proj_dir,
        profiles_dir=prof_dir
    )
)

# Compile SQL
compiled = compile_sql_code(ctx, sql)

# Execute SQL
execute_sql_code(ctx, sql)
```

### SQL Compilation Process

1. **Context Creation**: dbt project context is created with project and profiles directories
2. **Manifest Loading**: dbt manifest is parsed to understand project structure
3. **Jinja Compilation**: SQL with Jinja is compiled to plain SQL
4. **Reference Resolution**: `ref()`, `source()`, and other dbt functions are resolved
5. **Macro Expansion**: Custom macros are expanded
6. **Output**: Compiled SQL ready for execution

### Query Execution Process

1. **Connection**: Database connection is established using the selected target
2. **Execution**: Compiled SQL is sent to the database
3. **Result Fetching**: Results are fetched as a cursor
4. **Data Conversion**: Results are converted to pandas DataFrame
5. **Display**: Results are rendered in the UI

### Data Profiling Process

1. **Data Preparation**: Query results are converted to pandas DataFrame
2. **Profile Generation**: `ydata-profiling.ProfileReport` is created
3. **Report Conversion**: HTML report is generated
4. **Display**: HTML is rendered in the profiler panel

## Performance Considerations

### Caching

- **Manifest Caching**: dbt manifest is loaded once and cached
- **Profile Reports**: Profiling results are cached to avoid re-computation
- **Compiled SQL**: Compiled SQL is cached when possible

### State Management

- **Session State**: Streamlit's session state persists across re-renders
- **Minimal Re-renders**: Only necessary components are re-rendered
- **Efficient Updates**: State updates are optimized for performance

### Database Connections

- **Connection Pooling**: Database connections are managed by dbt adapters
- **Reuse Connections**: Connections are reused across queries
- **Cleanup**: Connections are properly closed

## Error Handling

The workbench implements robust error handling:

### Compilation Errors
```python
def compile(sql: str) -> str:
    try:
        return compile_sql_code(ctx, sql).compiled_code or ""
    except Exception as e:
        return str(e)  # Show error message in UI
```

### Execution Errors
```python
def run_query() -> None:
    try:
        resp, table = execute_sql_code(ctx, sql)
        # Process results
    except Exception as error:
        state.app.query_state = "error"
        state.app.query_adapter_resp = str(error)
        # Show error in UI
```

### General Errors
- Errors are caught and displayed in the UI
- Application state is preserved
- Users can continue working after errors

## Customization Options

### Query Templates

Users can customize the query template in the sidebar:

```sql
-- Default template
{sql}

-- Custom template with timing
EXPLAIN ANALYZE {sql}

-- Custom template with transaction
BEGIN TRANSACTION;
{sql}
ROLLBACK;
```

### Layout Customization

- Panels can be resized and repositioned
- Layout preferences can be saved
- Different layouts for different workflows

### Theme Support

- Light and dark mode support
- Custom CSS can be added
- Color schemes can be customized

## Advanced Features

### Hotkeys

The workbench supports keyboard shortcuts:

```python
event.Hotkey("ctrl+enter", sync(), bindInputs=True, overrideDefault=True)
event.Hotkey("command+s", sync(), bindInputs=True, overrideDefault=True)
event.Hotkey("ctrl+shift+enter", lambda: run_query(), bindInputs=True, overrideDefault=True)
event.Hotkey("command+shift+s", lambda: run_query(), bindInputs=True, overrideDefault=True)
```

### Dynamic Model Loading

Models are loaded dynamically from the dbt manifest:

```python
model_nodes = []
for node in app.ctx.manifest.nodes.values():
    if (
        node.resource_type == "model"
        and node.package_name == app.ctx.runtime_cfg.project_name
    ):
        model_nodes.append(node)
app.model_nodes = model_nodes
```

### RSS Feed Integration

Hacker News feed is fetched and displayed:

```python
hackernews_rss = feedparser.parse("https://news.ycombinator.com/rss")
feed_html = []
for entry in hackernews_rss.entries:
    feed_html.append(
        f"<div>...{entry.title}...{entry.published}...</div>"
    )
```

## Comparison with Other Tools

### VS Code + dbt Power User

| Feature | Workbench | VS Code + dbt Power User |
|---------|-----------|--------------------------|
| Real-time compilation | ✅ | ✅ |
| Query execution | ✅ | ✅ |
| Data profiling | ✅ | ❌ |
| Multiple targets | ✅ | ✅ |
| Model selection | ✅ | ❌ |
| Save to disk | ✅ | ✅ |
| Web-based | ✅ | ❌ |
| No VS Code required | ✅ | ❌ |
| Portable | ✅ | ❌ |

### dbt Cloud IDE

| Feature | Workbench | dbt Cloud IDE |
|---------|-----------|---------------|
| Free to use | ✅ | ❌ (paid) |
| Self-hosted | ✅ | ❌ |
| Open source | ✅ | ❌ |
| Customizable | ✅ | ❌ |
| Offline capable | ✅ | ❌ |
| Full dbt features | ✅ | ✅ |

## Future Enhancements

Potential improvements for the workbench:

1. **Collaboration Features**: Multi-user editing
2. **Version Control Integration**: Git integration
3. **Advanced Profiling**: More statistical analyses
4. **Query History**: Save and reuse queries
5. **Macro Testing**: Dedicated macro testing interface
6. **Documentation Preview**: View generated docs
7. **Test Runner**: Run dbt tests from workbench
8. **Schema Visualization**: Graph-based schema exploration
9. **Performance Analysis**: Query execution plans
10. **Team Features**: Shared layouts and preferences

## Conclusion

The dbt-osmosis workbench provides a powerful, interactive environment for dbt development by combining:

- **Real-time compilation** for instant feedback
- **Query execution** against multiple targets
- **Data profiling** for better understanding
- **Modular architecture** for extensibility
- **Streamlit-based UI** for ease of use

This creates a seamless development experience that accelerates dbt development and improves productivity.
