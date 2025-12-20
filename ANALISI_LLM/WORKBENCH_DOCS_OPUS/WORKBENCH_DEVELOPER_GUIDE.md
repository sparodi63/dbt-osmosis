# dbt-osmosis Workbench Developer Guide

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Technology Stack](#2-technology-stack)
3. [Module Structure](#3-module-structure)
4. [Component Deep Dive](#4-component-deep-dive)
5. [State Management](#5-state-management)
6. [dbt Integration](#6-dbt-integration)
7. [Adding New Components](#7-adding-new-components)
8. [Extending Functionality](#8-extending-functionality)
9. [Testing](#9-testing)
10. [Deployment](#10-deployment)

---

## 1. Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLI Layer                                       │
│                         (cli/main.py:workbench)                             │
│    Parses args → Launches Streamlit subprocess → Passes config              │
└────────────────────────────────────────────────────────────────┬────────────┘
                                                                 │
                                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Streamlit Application                              │
│                           (workbench/app.py)                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         Session State (state.app)                        ││
│  │  ctx, model, query, compiled_query, query_result_df, profile_html, ...  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                    │                                         │
│            ┌───────────────────────┼───────────────────────┐                │
│            │                       │                       │                │
│            ▼                       ▼                       ▼                │
│    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐         │
│    │   Sidebar     │      │   Dashboard   │      │   Elements    │         │
│    │  - Models     │      │  (Grid Layout)│      │  (Hotkeys)    │         │
│    │  - Profiles   │      │               │      │               │         │
│    │  - Template   │      │               │      │               │         │
│    └───────────────┘      └───────┬───────┘      └───────────────┘         │
│                                   │                                         │
│         ┌─────────────┬───────────┼───────────┬─────────────┐              │
│         │             │           │           │             │              │
│         ▼             ▼           ▼           ▼             ▼              │
│    ┌─────────┐  ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐         │
│    │ Editor  │  │ Renderer │ │ Preview │ │ Profiler │ │  Feed   │         │
│    │         │  │          │ │         │ │          │ │         │         │
│    │ Monaco  │  │  Monaco  │ │ DataGrid│ │  Iframe  │ │  HTML   │         │
│    │ Editor  │  │ ReadOnly │ │  Table  │ │  Report  │ │  Feed   │         │
│    └────┬────┘  └──────────┘ └────┬────┘ └────┬─────┘ └─────────┘         │
│         │                         │           │                            │
└─────────┼─────────────────────────┼───────────┼────────────────────────────┘
          │                         │           │
          ▼                         ▼           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           dbt-osmosis Core                                   │
│  ┌───────────────────────┐  ┌────────────────────┐  ┌────────────────────┐ │
│  │ compile_sql_code()    │  │ execute_sql_code() │  │ DbtProjectContext  │ │
│  │ Jinja → SQL           │  │ SQL → Results      │  │ Manifest, Adapter  │ │
│  └───────────────────────┘  └────────────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Data Warehouse                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Input (SQL)
    │
    ▼
Editor Component
    │ onChange
    ▼
compile() function
    │ compile_sql_code()
    ▼
state.app.compiled_query
    │
    ▼
Renderer Component (displays compiled SQL)
    │
    │ User clicks "Run Query"
    ▼
run_query() function
    │ execute_sql_code()
    ▼
state.app.query_result_df
    │
    ▼
Preview Component (displays DataGrid)
    │
    │ User clicks "Profile"
    ▼
run_profile() function
    │ ydata_profiling
    ▼
state.app.profile_html
    │
    ▼
Profiler Component (displays iframe)
```

---

## 2. Technology Stack

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `streamlit` | 1.20.0 - 1.33.x | Web application framework |
| `streamlit-elements-fluence` | >= 0.1.4 | Dashboard components, Material UI |
| `streamlit-ace` | ~0.1.1 | Code editor (Monaco) |
| `ydata-profiling` | ~4.12.1 | Data profiling reports |
| `feedparser` | ~6.0.11 | RSS feed parsing |
| `pandas` | (transitive) | Data manipulation |
| `dbt-core` | 1.8 - 1.10.x | dbt integration |

### Key Libraries Explained

#### streamlit-elements-fluence

Fork of `streamlit-elements` providing:
- `dashboard.Grid` - Draggable/resizable grid layout
- `mui` - Material UI components
- `editor.Monaco` - VS Code-style editor
- `html.Iframe` - Embedded HTML content
- `event.Hotkey` - Keyboard shortcuts
- `sync()` - State synchronization trigger

#### ydata-profiling

Generates HTML statistical reports:
- Column analysis
- Distributions
- Correlations
- Missing values

---

## 3. Module Structure

### File Tree

```
src/dbt_osmosis/workbench/
├── __init__.py                 # Empty (package marker)
├── app.py                      # Main Streamlit application (372 lines)
└── components/
    ├── __init__.py             # Empty
    ├── dashboard.py            # Dashboard grid & base Item class (83 lines)
    ├── editor.py               # SQL/YAML editor component (119 lines)
    ├── renderer.py             # Compiled SQL display (58 lines)
    ├── preview.py              # Query results DataGrid (78 lines)
    ├── profiler.py             # Data profiler iframe (57 lines)
    ├── feed.py                 # RSS feed display (43 lines)
    └── renderer.py             # Compiled SQL renderer (58 lines)
```

### Module Responsibilities

| File | Responsibility |
|------|----------------|
| `app.py` | Application entry point, state initialization, sidebar, callbacks |
| `dashboard.py` | Abstract `Dashboard.Item` base class, grid management, theming |
| `editor.py` | Monaco editor with SQL/YAML tabs, compilation trigger |
| `renderer.py` | Read-only Monaco showing compiled SQL |
| `preview.py` | MUI DataGrid for query results, query execution trigger |
| `profiler.py` | Iframe displaying ydata-profiling HTML report |
| `feed.py` | Iframe displaying RSS feed (Hacker News) |

---

## 4. Component Deep Dive

### 4.1 Dashboard (dashboard.py)

The `Dashboard` class manages the grid layout system.

#### Class: `Dashboard`

```python
class Dashboard:
    DRAGGABLE_CLASS = "draggable"  # CSS class for drag handles

    def __init__(self) -> None:
        self._layout: list[Dashboard.Item] = []  # Registered items

    def register(self, item: Dashboard.Item) -> None:
        """Register an item to the dashboard layout."""
        self._layout.append(item)

    @contextmanager
    def __call__(self, **props) -> Generator[None, None, None]:
        """Context manager for rendering the grid."""
        props["draggableHandle"] = f".{Dashboard.DRAGGABLE_CLASS}"
        with dashboard.Grid(self._layout, **props):
            yield
```

#### Abstract Class: `Dashboard.Item`

All dashboard components inherit from this:

```python
class Item(ABC):
    @staticmethod
    def initial_state() -> dict[str, Any]:
        """Return initial state keys for session state."""
        return {}

    def __init__(self, board: Dashboard, x: int, y: int, w: int, h: int, **item_props):
        self._key: str = str(uuid4())           # Unique component key
        self._draggable_class: str = Dashboard.DRAGGABLE_CLASS
        self._dark_mode: bool = DarkMode()      # Theme state
        board.register(dashboard.Item(self._key, x, y, w, h, **item_props))

    @contextmanager
    def title_bar(self, padding: str, dark_switcher: bool = True):
        """Render a standard title bar with optional theme switcher."""
        with mui.Stack(...):
            yield
            if dark_switcher:
                # Theme toggle button

    @abstractmethod
    def __call__(self) -> None:
        """Render the component. Must be implemented."""
        raise NotImplementedError
```

### 4.2 Editor Component (editor.py)

The main code editor with SQL and YAML tabs.

#### Key Features

```python
class Editor(Dashboard.Item):
    def __init__(self, *args, compile_action: Callable[[str], str], **kwargs):
        self.tabs = {
            TabName.SQL: {"content": "", "language": "sql"},
            TabName.YAML: {"content": "version: 2\n", "language": "yaml"},
        }
        self._compile_action = compile_action   # Callback for compilation
        self._realtime_compilation = False       # Toggle for auto-compile

    def update_content(self, tab_name: TabName, content: str) -> None:
        """Update tab content and trigger compilation for SQL."""
        self.tabs[tab_name]["content"] = content
        if tab_name == TabName.SQL:
            state.app.compiled_query = self._compile_action(content)

    def __call__(self):
        # Renders:
        # 1. Title bar with tabs
        # 2. Monaco editor for each tab
        # 3. Compile button and realtime toggle
```

#### Monaco Editor Integration

```python
editor.Monaco(
    css={"padding": "0 2px 0 2px"},
    defaultValue=tab["content"],
    language=tab["language"],           # "sql" or "yaml"
    onChange=on_change,                  # Callback on edit
    theme="vs-dark" if self._dark_mode else "light",
    path=path,                           # For editor state
    options=props,
)
```

### 4.3 Renderer Component (renderer.py)

Displays the compiled SQL in a read-only Monaco editor.

```python
class Renderer(Dashboard.Item):
    def __call__(self):
        # Read-only Monaco showing state.app.compiled_query
        editor.Monaco(
            defaultValue=state.app.compiled_query or "",
            language="sql",
            options={"readOnly": True},
            key=md5(state.app.compiled_query.encode()).hexdigest(),
        )
```

**Note**: The `key` is a hash of the content to force re-render when content changes.

### 4.4 Preview Component (preview.py)

Displays query results in a Material UI DataGrid.

```python
class Preview(Dashboard.Item):
    @staticmethod
    def initial_state() -> dict[str, Any]:
        return {
            "query_adapter_resp": None,
            "query_result_df": pd.DataFrame(),
            "query_result_columns": [],
            "query_result_rows": [],
            "query_state": "test",
            "query_template": "select * from ({sql}) as _query limit 200",
        }

    def __call__(self):
        # Shows different UI based on query_state:
        # - "running": CircularProgress spinner
        # - "error": Error message
        # - "success": DataGrid with results
        # - default: "No results" message

        mui.DataGrid(
            columns=state.app.query_result_columns,
            rows=state.app.query_result_rows,
            pageSize=20,
            getRowId=JSCallback("(row) => Math.random()"),
        )
```

### 4.5 Profiler Component (profiler.py)

Embeds the ydata-profiling HTML report in an iframe.

```python
class Profiler(Dashboard.Item):
    def __call__(self):
        if state.app.profile_html:
            # Show generated profile
            html.Iframe(
                srcDoc=state.app.profile_html,
                style={"width": "100%", "height": "100%"},
            )
        else:
            # Show documentation as placeholder
            html.Iframe(
                src="https://pandas-profiling.github.io/...",
            )
```

### 4.6 Feed Component (feed.py)

Displays an RSS feed (Hacker News by default).

```python
class RssFeed(Dashboard.Item):
    def __call__(self):
        extras.InnerHTML(state.app.feed_html)
```

The feed HTML is generated during initialization in `app.py`:

```python
hackernews_rss = feedparser.parse("https://news.ycombinator.com/rss")
feed_html = []
for entry in hackernews_rss.entries:
    feed_html.append(f"""
        <div>
            <a href="{entry.link}">{entry.title}</a>
            <div>{entry.published}</div>
        </div>
    """)
state.app.feed_html = "".join(feed_html)
```

---

## 5. State Management

### Streamlit Session State

All application state is stored in `st.session_state.app` (aliased as `state.app`).

#### State Initialization (app.py)

```python
if "app" not in state:
    board = Dashboard()

    app = SimpleNamespace(
        # Model selection
        model="SCRATCH",
        model_nodes=[],

        # Dashboard components
        dashboard=board,
        editor=Editor(board, 0, 0, 6, 11, compile_action=compile),
        renderer=Renderer(board, 6, 0, 6, 11),
        preview=Preview(board, 0, 11, 12, 9, query_action=run_query),
        profiler=Profiler(board, 0, 20, 8, 9, prof_action=run_profile),
        feed=RssFeed(board, 8, 20, 4, 9),
    )

    # Collect initial state from all components
    for v in vars(app).values():
        if isinstance(v, Dashboard.Item):
            for k, v in v.initial_state().items():
                setattr(app, k, v)

    state.app = app
```

#### State Keys

| Key | Type | Description |
|-----|------|-------------|
| `model` | `str` or `ModelNode` | Currently selected model |
| `model_nodes` | `list[ModelNode]` | All available models |
| `ctx` | `DbtProjectContext` | dbt project context |
| `target_name` | `str` | Current dbt target |
| `query` | `str` | Raw SQL in editor |
| `compiled_query` | `str` | Compiled SQL output |
| `query_state` | `str` | "test", "running", "success", "error" |
| `query_template` | `str` | SQL wrapper template |
| `query_result_df` | `DataFrame` | Query results |
| `query_result_columns` | `list[dict]` | DataGrid columns |
| `query_result_rows` | `list[dict]` | DataGrid rows |
| `query_adapter_resp` | `str` | Adapter response message |
| `profile_html` | `str` | Profiler HTML report |
| `feed_html` | `str` | RSS feed HTML |
| `theme` | `str` | "dark" or "light" |
| `all_profiles` | `dict` | All dbt profiles |

### State Flow Diagram

```
┌─────────────┐     onChange      ┌──────────────────┐
│   Editor    │ ─────────────────▶│ state.app.query  │
└─────────────┘                   └────────┬─────────┘
                                           │
                                           ▼ compile()
                                  ┌──────────────────────┐
                                  │state.app.compiled_   │
                                  │         query        │
                                  └────────┬─────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
           ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
           │  Renderer   │        │   Preview   │        │  Profiler   │
           │  (display)  │        │ (run_query) │        │(run_profile)│
           └─────────────┘        └──────┬──────┘        └──────┬──────┘
                                         │                      │
                                         ▼                      ▼
                               ┌─────────────────┐    ┌─────────────────┐
                               │query_result_df  │    │ profile_html    │
                               │query_result_*   │    │                 │
                               └─────────────────┘    └─────────────────┘
```

---

## 6. dbt Integration

### Core Functions Used

The workbench uses these functions from `dbt_osmosis.core.osmosis`:

```python
from dbt_osmosis.core.osmosis import (
    DbtConfiguration,           # Configuration dataclass
    create_dbt_project_context, # Initialize dbt environment
    compile_sql_code,           # Compile Jinja SQL
    execute_sql_code,           # Execute SQL against warehouse
    _reload_manifest,           # Refresh manifest after changes
    discover_profiles_dir,      # Find profiles.yml
    discover_project_dir,       # Find dbt_project.yml
)
```

### Compilation Flow

```python
def compile(sql: str) -> str:
    """Compile SQL using dbt context."""
    ctx: DbtProject = state.app.ctx
    try:
        return compile_sql_code(ctx, sql).compiled_code or ""
    except Exception as e:
        return str(e)  # Return error as string for display
```

### Query Execution Flow

```python
def run_query() -> None:
    """Run SQL query using dbt context."""
    ctx: DbtProject = state.app.ctx
    sql = state.app.compiled_query

    try:
        state.app.query_state = "running"
        # Execute with template wrapper
        resp, table = execute_sql_code(
            ctx,
            state.app.query_template.format(sql=sql)
        )
    except Exception as error:
        state.app.query_state = "error"
        state.app.query_adapter_resp = str(error)
    else:
        state.app.query_state = "success"
        state.app.query_adapter_resp = resp
        # Convert agate table to DataFrame
        state.app.query_result_df = pd.DataFrame(...)
```

### Target Switching

```python
def change_target() -> None:
    """Change the target profile."""
    ctx: DbtProject = state.app.ctx
    if ctx.runtime_cfg.target_name != state.app.target_name:
        ctx.runtime_cfg.target_name = state.app.target_name
        _reload_manifest(ctx)  # Refresh for new target
        state.app.compiled_query = compile(state.app.query)
```

---

## 7. Adding New Components

### Step 1: Create Component File

Create a new file in `workbench/components/`:

```python
# components/my_component.py
import typing as t
from streamlit import session_state as state
from streamlit_elements_fluence import mui
from .dashboard import Dashboard

@t.final
class MyComponent(Dashboard.Item):
    @staticmethod
    def initial_state() -> dict[str, t.Any]:
        """Define initial state keys."""
        return {
            "my_data": None,
            "my_loading": False,
        }

    def __init__(self, *args, my_callback: t.Callable, **kwargs) -> None:
        super().__init__(*args, **kwargs)
        self._my_callback = my_callback

    def __call__(self, **props) -> None:
        """Render the component."""
        with mui.Paper(
            key=self._key,
            sx={
                "display": "flex",
                "flexDirection": "column",
                "borderRadius": 3,
                "overflow": "hidden",
            },
            elevation=1,
        ):
            # Title bar
            with self.title_bar():
                mui.icon.MyIcon()
                mui.Typography("My Component")

            # Content area
            with mui.Box(sx={"flex": 1, "minHeight": 0}):
                if state.app.my_loading:
                    mui.CircularProgress()
                elif state.app.my_data:
                    mui.Typography(str(state.app.my_data))
                else:
                    mui.Typography("No data")

            # Action bar
            with mui.Stack(direction="row", spacing=2, sx={"padding": "10px"}):
                mui.Button(
                    "Action",
                    variant="contained",
                    onClick=lambda: self._my_callback(),
                )
```

### Step 2: Register in app.py

```python
from dbt_osmosis.workbench.components.my_component import MyComponent

# Define callback
def my_action() -> None:
    state.app.my_loading = True
    # ... do work ...
    state.app.my_data = result
    state.app.my_loading = False

# In main() initialization
app = SimpleNamespace(
    # ... existing components ...
    my_component=MyComponent(
        board,
        x=0, y=29, w=6, h=5,  # Grid position
        my_callback=my_action
    ),
)

# In main() rendering
with app.dashboard(rowHeight=57):
    # ... existing components ...
    app.my_component()
```

### Step 3: Export (Optional)

Add to `components/__init__.py`:

```python
from .my_component import MyComponent

__all__ = [..., "MyComponent"]
```

---

## 8. Extending Functionality

### Adding Keyboard Shortcuts

In `app.py`, within the `elements("dashboard")` context:

```python
with elements("dashboard"):
    # Existing shortcuts
    event.Hotkey("ctrl+enter", sync(), bindInputs=True)

    # Add new shortcut
    event.Hotkey("ctrl+shift+p", lambda: my_action(), bindInputs=True)
```

### Adding Sidebar Sections

```python
def sidebar(ctx: DbtProject) -> None:
    # ... existing sections ...

    with st.sidebar.expander("🔧 My Section", expanded=False):
        st.caption("Description of this section")
        state.app.my_setting = st.selectbox(
            "Choose option",
            options=["Option A", "Option B"],
            on_change=my_handler,
        )
```

### Custom Data Converters

For handling special data types in query results:

```python
def make_json_compat(v: Any) -> Any:
    """Convert a value to be safe for JSON serialization."""
    if isinstance(v, decimal.Decimal):
        return float(v)
    if isinstance(v, date):
        return v.isoformat()
    if isinstance(v, datetime):
        return v.isoformat()
    # Add custom types here
    if isinstance(v, MyCustomType):
        return str(v)
    return v
```

### Adding New Tabs to Editor

In `editor.py`:

```python
class TabName(str, Enum):
    SQL = "SQL"
    YAML = "YAML"
    CONFIG = "CONFIG"  # New tab

class Editor(Dashboard.Item):
    def __init__(self, ...):
        self.tabs = {
            TabName.SQL: {"content": "", "language": "sql"},
            TabName.YAML: {"content": "version: 2\n", "language": "yaml"},
            TabName.CONFIG: {"content": "{}", "language": "json"},  # New
        }
```

---

## 9. Testing

### Running Locally

```bash
cd /path/to/dbt-osmosis

# Install in development mode
pip install -e ".[workbench,dev]"

# Run from source
python -m streamlit run src/dbt_osmosis/workbench/app.py -- \
  --project-dir /path/to/test/project
```

### Manual Testing Checklist

- [ ] Application starts without errors
- [ ] Models load in sidebar
- [ ] SQL compilation works
- [ ] Target switching works
- [ ] Query execution returns results
- [ ] DataGrid displays correctly
- [ ] Profiler generates reports
- [ ] Theme switching works
- [ ] Keyboard shortcuts work
- [ ] Save/Revert model works

### Unit Testing Components

Since Streamlit components are tightly coupled to the framework, testing typically involves:

1. **Logic Testing**: Extract business logic into separate functions
2. **Integration Testing**: Use Streamlit's testing utilities
3. **E2E Testing**: Selenium/Playwright for UI testing

```python
# Example: Test compilation logic
def test_compile_returns_error_on_invalid_sql():
    # Mock the context
    ctx = MockDbtContext()
    result = compile_sql_code(ctx, "select * from {{ ref('nonexistent') }}")
    assert "error" in result.lower()
```

---

## 10. Deployment

### Streamlit Cloud

1. Create `requirements.txt`:
```
dbt-osmosis[workbench]
dbt-duckdb  # or your adapter
```

2. Add `.streamlit/config.toml`:
```toml
[server]
headless = true
port = 8501

[browser]
gatherUsageStats = false
```

3. Deploy via Streamlit Cloud dashboard

### Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install "dbt-osmosis[workbench]" dbt-duckdb

# Copy dbt project
COPY . /app/dbt_project

WORKDIR /app/dbt_project

# Expose port
EXPOSE 8501

# Run workbench
CMD ["dbt-osmosis", "workbench", "--host", "0.0.0.0"]
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `DBT_PROJECT_DIR` | Path to dbt project |
| `DBT_PROFILES_DIR` | Path to profiles.yml |
| `DBT_TARGET` | Default target |
| Standard dbt adapter vars | `SNOWFLAKE_ACCOUNT`, etc. |

---

## Appendix A: Class Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          Dashboard                               │
├─────────────────────────────────────────────────────────────────┤
│ - _layout: list[Dashboard.Item]                                 │
├─────────────────────────────────────────────────────────────────┤
│ + register(item: Item)                                          │
│ + __call__(**props) -> ContextManager                           │
└─────────────────────────────────────────────────────────────────┘
                              △
                              │ contains
                              │
┌─────────────────────────────────────────────────────────────────┐
│                     Dashboard.Item (ABC)                         │
├─────────────────────────────────────────────────────────────────┤
│ - _key: str                                                     │
│ - _draggable_class: str                                         │
│ - _dark_mode: bool                                              │
├─────────────────────────────────────────────────────────────────┤
│ + initial_state() -> dict  [static]                             │
│ + title_bar() -> ContextManager                                 │
│ + __call__() [abstract]                                         │
└─────────────────────────────────────────────────────────────────┘
          △               △               △               △
          │               │               │               │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
    │  Editor   │   │ Renderer  │   │  Preview  │   │ Profiler  │
    ├───────────┤   ├───────────┤   ├───────────┤   ├───────────┤
    │ tabs      │   │           │   │_query_    │   │_prof_     │
    │_compile_  │   │           │   │ action    │   │ action    │
    │ action    │   │           │   │           │   │           │
    └───────────┘   └───────────┘   └───────────┘   └───────────┘
```

---

## Appendix B: Key File Locations

| Purpose | Location |
|---------|----------|
| Main app entry | `src/dbt_osmosis/workbench/app.py` |
| Components | `src/dbt_osmosis/workbench/components/` |
| CLI command | `src/dbt_osmosis/cli/main.py:workbench()` |
| dbt integration | `src/dbt_osmosis/core/osmosis.py` |
| SQL operations | `src/dbt_osmosis/core/sql_operations.py` |

---

*Developer Guide for dbt-osmosis Workbench v1.1.17*
*Last Updated: December 2024*
