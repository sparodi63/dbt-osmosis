# dbt-osmosis: Complete Technical Documentation

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Core Modules Detailed Reference](#3-core-modules-detailed-reference)
4. [CLI Commands Reference](#4-cli-commands-reference)
5. [Data Flow Diagrams](#5-data-flow-diagrams)
6. [Configuration Reference](#6-configuration-reference)
7. [Plugin System](#7-plugin-system)
8. [LLM Integration](#8-llm-integration)
9. [Workbench (Streamlit UI)](#9-workbench-streamlit-ui)
10. [Maintenance Guide](#10-maintenance-guide)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Executive Summary

### 1.1 What is dbt-osmosis?

**dbt-osmosis** is a CLI utility and library that extends dbt (data build tool) functionality by providing automated management of YAML schema documentation files. It solves the common pain point of maintaining synchronization between actual database schema and dbt YAML documentation.

### 1.2 Core Capabilities

| Feature | Description |
|---------|-------------|
| **YAML Refactoring** | Automatically restructures and organizes schema YAML files based on configurable path rules |
| **Documentation Inheritance** | Propagates column descriptions, tags, and metadata from upstream sources to downstream models |
| **Database Introspection** | Queries the data warehouse to discover actual column names, types, and synchronize with YAML |
| **LLM Documentation Synthesis** | Uses AI (OpenAI, Azure OpenAI, Anthropic, etc.) to generate missing column/table descriptions |
| **Workbench UI** | Streamlit-based interactive development environment for dbt SQL |
| **SQL Proxy** | MySQL-compatible proxy server for BI tool integration |

### 1.3 Version & Requirements

- **Current Version**: 1.1.17
- **Python**: 3.10, 3.11, 3.12
- **dbt-core**: 1.8 - 1.10.x
- **License**: Apache-2.0

---

## 2. Architecture Overview

### 2.1 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              dbt-osmosis CLI                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        Command Layer (cli/main.py)                      ││
│  │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌─────────────┐  ││
│  │   │  yaml    │ │   yaml   │ │   yaml   │ │ workbench │ │  sql run/   │  ││
│  │   │ refactor │ │ organize │ │ document │ │           │ │   compile   │  ││
│  │   └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬─────┘ └──────┬──────┘  ││
│  └────────┼────────────┼────────────┼─────────────┼──────────────┼─────────┘│
│           │            │            │             │              │          │
│  ┌────────▼────────────▼────────────▼─────────────┼──────────────┼─────────┐│
│  │                    Core Layer (core/*)                        │         ││
│  │  ┌──────────────┐  ┌─────────────────┐  ┌────────────────┐    │         ││
│  │  │   config.py  │  │   settings.py   │  │   logger.py    │    │         ││
│  │  │ DbtConfig    │  │ YamlRefactor    │  │ Rich Console   │    │         ││
│  │  │ DbtContext   │  │ Settings/Ctx    │  │ File Logging   │    │         ││
│  │  └──────┬───────┘  └────────┬────────┘  └────────────────┘    │         ││
│  │         │                   │                                  │         ││
│  │  ┌──────▼───────────────────▼────────────────────────────────┐│         ││
│  │  │              Transform Pipeline (transforms.py)           ││         ││
│  │  │  ┌────────────────┐  ┌─────────────────┐  ┌────────────┐  ││         ││
│  │  │  │ inject_missing │─▶│ remove_extra    │─▶│  inherit   │  ││         ││
│  │  │  │ _columns       │  │ _columns        │  │ _knowledge │  ││         ││
│  │  │  └────────────────┘  └─────────────────┘  └──────┬─────┘  ││         ││
│  │  │                                                   │        ││         ││
│  │  │  ┌────────────────┐  ┌─────────────────┐  ┌──────▼─────┐  ││         ││
│  │  │  │  synchronize   │◀─│ sort_columns    │◀─│ synthesize │  ││         ││
│  │  │  │  _data_types   │  │ _as_configured  │  │  _with_llm │  ││         ││
│  │  │  └────────────────┘  └─────────────────┘  └────────────┘  ││         ││
│  │  └───────────────────────────────────────────────────────────┘│         ││
│  │                                                                │         ││
│  │  ┌───────────────────────────────────────────────────────────┐│         ││
│  │  │            Supporting Modules                              ││         ││
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ ││         ││
│  │  │  │ inheritance  │  │introspection │  │ restructuring    │ ││         ││
│  │  │  │ .py          │  │.py           │  │ .py              │ ││         ││
│  │  │  └──────────────┘  └──────────────┘  └──────────────────┘ ││         ││
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ ││         ││
│  │  │  │ node_filters │  │path_mgmt     │  │ sync_operations  │ ││         ││
│  │  │  │ .py          │  │.py           │  │ .py              │ ││         ││
│  │  │  └──────────────┘  └──────────────┘  └──────────────────┘ ││         ││
│  │  └───────────────────────────────────────────────────────────┘│         ││
│  └───────────────────────────────────────────────────────────────┴─────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                      Schema Layer (core/schema/*)                        ││
│  │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              ││
│  │   │  parser.py   │    │  reader.py   │    │  writer.py   │              ││
│  │   │ruamel config │    │ YAML cache   │    │ disk commit  │              ││
│  │   └──────────────┘    └──────────────┘    └──────────────┘              ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                      External Integrations                               ││
│  │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              ││
│  │   │   llm.py     │    │  plugins.py  │    │sql/proxy.py  │              ││
│  │   │ OpenAI/Azure │    │ pluggy hooks │    │MySQL proxy   │              ││
│  │   └──────────────┘    └──────────────┘    └──────────────┘              ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           dbt-core Integration                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │RuntimeConfig │  │  Manifest    │  │   Adapter    │  │  Parsers     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Data Warehouse                                      │
│      (Snowflake, BigQuery, Redshift, PostgreSQL, DuckDB, etc.)               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Directory Structure

```
src/dbt_osmosis/
├── __init__.py              # Empty package init
├── __main__.py              # Entry point: python -m dbt_osmosis
├── cli/
│   ├── __init__.py
│   └── main.py              # CLI commands (Click framework) - 658 lines
├── core/
│   ├── __init__.py          # Exports all core functions
│   ├── osmosis.py           # Re-exports for backwards compatibility
│   ├── config.py            # DbtConfiguration, DbtProjectContext - 275 lines
│   ├── settings.py          # YamlRefactorSettings, YamlRefactorContext - 184 lines
│   ├── logger.py            # Rich console + file logging - 97 lines
│   ├── transforms.py        # Transform pipeline & operations - 548 lines
│   ├── inheritance.py       # Column documentation inheritance - 200 lines
│   ├── introspection.py     # Database schema introspection - 311 lines
│   ├── restructuring.py     # YAML file reorganization - 344 lines
│   ├── node_filters.py      # Node filtering & topological sort - 151 lines
│   ├── path_management.py   # YAML path resolution - 250 lines
│   ├── sync_operations.py   # Node → YAML synchronization - 252 lines
│   ├── plugins.py           # Pluggy plugin system - 71 lines
│   ├── llm.py               # LLM doc synthesis - 488 lines
│   ├── sql_operations.py    # SQL compile/execute - 65 lines
│   └── schema/
│       ├── __init__.py
│       ├── parser.py        # ruamel.yaml configuration
│       ├── reader.py        # Cached YAML reading
│       └── writer.py        # Batched YAML writing
├── sql/
│   └── proxy.py             # MySQL proxy for BI tools
└── workbench/
    ├── app.py               # Streamlit main app
    └── components/
        ├── dashboard.py     # Layout management
        ├── editor.py        # SQL/YAML editor
        ├── preview.py       # Query result preview
        ├── profiler.py      # Data profiling
        ├── feed.py          # RSS feed
        └── renderer.py      # Rendering utilities
```

### 2.3 Key Design Patterns

| Pattern | Implementation | Purpose |
|---------|---------------|---------|
| **Dataclass Configuration** | `DbtConfiguration`, `YamlRefactorSettings` | Type-safe, immutable configuration objects |
| **Context Object** | `DbtProjectContext`, `YamlRefactorContext` | Single object passed through all operations containing shared state |
| **Transform Pipeline** | `TransformOperation >> TransformOperation` | Chainable operations with `>>` operator |
| **Plugin Architecture** | `pluggy` hooks in `plugins.py` | Extensible column name matching |
| **Lazy Loading** | Manifest, catalog, adapter | Resources loaded on-demand to reduce startup time |
| **Thread-safe Caching** | `_YAML_BUFFER_CACHE`, `_COLUMN_LIST_CACHE` | Avoid redundant I/O with mutex protection |
| **Dry Run Support** | All operations respect `dry_run` flag | Safe testing without disk modifications |

---

## 3. Core Modules Detailed Reference

### 3.1 config.py - dbt Project Configuration

**Location**: `src/dbt_osmosis/core/config.py` (275 lines)

#### Classes

##### `DbtConfiguration`
```python
@dataclass
class DbtConfiguration:
    project_dir: str          # Path to dbt_project.yml directory
    profiles_dir: str         # Path to profiles.yml directory
    target: str | None        # dbt target to use
    profile: str | None       # dbt profile to use
    threads: int | None       # Number of threads
    single_threaded: bool     # Force single-threaded mode
    vars: dict[str, Any]      # dbt variables
    quiet: bool               # Suppress output
    disable_introspection: bool  # Skip database queries
```

##### `DbtProjectContext`
```python
@dataclass
class DbtProjectContext:
    config: DbtConfiguration     # Original configuration
    runtime_cfg: RuntimeConfig   # dbt RuntimeConfig
    manifest: Manifest           # Parsed dbt manifest
    sql_parser: SqlBlockParser   # SQL compilation parser
    macro_parser: SqlMacroParser # Macro parser
    connection_ttl: float        # Max connection lifetime (default: 3600s)

    @property
    def adapter(self) -> BaseAdapter:  # Lazy-loaded database adapter

    @property
    def is_connection_expired(self) -> bool:  # Check TTL
```

#### Key Functions

| Function | Description |
|----------|-------------|
| `discover_project_dir()` | Finds `dbt_project.yml` checking `DBT_PROJECT_DIR` env, CWD, and parents |
| `discover_profiles_dir()` | Finds `profiles.yml` checking `DBT_PROFILES_DIR` env, CWD, then `~/.dbt` |
| `create_dbt_project_context(config)` | Initializes full dbt environment with manifest, adapter, parsers |
| `config_to_namespace(cfg)` | Converts `DbtConfiguration` to `argparse.Namespace` for dbt compatibility |
| `_reload_manifest(context)` | Thread-safe manifest reloading after YAML changes |

#### dbt-loom Integration

The module supports **dbt-loom** for cross-project references. If dbt-loom is installed, it automatically:
1. Loads external manifests defined in loom configuration
2. Adds exposed nodes from other projects to the local manifest
3. Enables cross-project documentation inheritance

---

### 3.2 settings.py - Refactoring Settings & Context

**Location**: `src/dbt_osmosis/core/settings.py` (184 lines)

#### Classes

##### `YamlRefactorSettings`
```python
@dataclass
class YamlRefactorSettings:
    # Filtering
    fqn: list[str]                    # Filter by fully qualified name
    models: list[Path | str]          # Filter by file path

    # Operation modes
    dry_run: bool                     # Don't write to disk

    # Skip flags
    skip_merge_meta: bool             # Don't merge upstream meta
    skip_add_columns: bool            # Don't add missing columns
    skip_add_tags: bool               # Don't add upstream tags
    skip_add_data_types: bool         # Don't add data types
    skip_add_source_columns: bool     # Don't add source columns

    # Enhancement flags
    add_progenitor_to_meta: bool      # Track column origin
    numeric_precision_and_scale: bool # Use precise numeric types
    string_length: bool               # Include string lengths
    force_inherit_descriptions: bool  # Override existing descriptions
    use_unrendered_descriptions: bool # Preserve {{ doc(...) }} blocks
    add_inheritance_for_specified_keys: list[str]  # Extra keys to inherit
    output_to_lower: bool             # Lowercase output

    # Catalog
    catalog_path: str | None          # Path to catalog.json
    create_catalog_if_not_exists: bool
```

##### `YamlRefactorContext`
```python
@dataclass
class YamlRefactorContext:
    project: DbtProjectContext        # dbt project context
    settings: YamlRefactorSettings    # Refactoring settings
    pool: ThreadPoolExecutor          # Thread pool for parallel ops
    yaml_handler: ruamel.yaml.YAML    # YAML parser instance
    yaml_handler_lock: threading.Lock # Thread safety

    placeholders: tuple[str, ...]     # Undocumented markers:
                                      # ("", "Pending...", "Not documented", etc.)

    @property
    def mutation_count(self) -> int   # Number of changes made

    @property
    def mutated(self) -> bool         # Were any changes made?

    @property
    def source_definitions(self) -> dict  # From dbt_project.yml vars

    @property
    def ignore_patterns(self) -> list[str]  # Column skip patterns
```

---

### 3.3 transforms.py - Transform Pipeline

**Location**: `src/dbt_osmosis/core/transforms.py` (548 lines)

This is the heart of dbt-osmosis. It implements a chainable transformation system.

#### Transform Operation System

```python
@dataclass
class TransformOperation:
    func: Callable          # The transformation function
    name: str               # Display name

    def __call__(self, context, node=None):
        # Execute the operation

    def __rshift__(self, next_op):
        # Chain: op1 >> op2 creates TransformPipeline
```

```python
@dataclass
class TransformPipeline:
    operations: list[TransformOperation]
    commit_mode: Literal["none", "batch", "atomic", "defer"]

    def __call__(self, context, node=None):
        # Execute all operations in sequence
        # Commit YAML changes based on commit_mode
```

#### Available Transforms

##### `inject_missing_columns`
```
Purpose: Add columns found in database but missing from YAML
Flow:
  1. Get columns from database/catalog (get_columns)
  2. Compare with node.columns
  3. Add missing columns with empty description
Respects: skip_add_columns, skip_add_source_columns settings
```

##### `remove_columns_not_in_database`
```
Purpose: Remove YAML columns that don't exist in database
Flow:
  1. Get actual columns from database
  2. Find columns in YAML not in database
  3. Remove orphaned columns
Note: Protects against accidental documentation for non-existent columns
```

##### `inherit_upstream_column_knowledge`
```
Purpose: Propagate documentation from ancestors to descendants
Flow:
  1. Build ancestor tree (generation_0, generation_1, ...)
  2. Build column knowledge graph
  3. Merge descriptions, tags, meta from ancestors
  4. Respect force_inherit_descriptions setting
Inherits: description, tags, meta, custom keys
```

##### `sort_columns_as_in_database`
```
Purpose: Reorder YAML columns to match database order
Uses: Column index from introspection
```

##### `sort_columns_alphabetically`
```
Purpose: Alphabetically sort columns in YAML
```

##### `sort_columns_as_configured`
```
Purpose: Sort using per-model configuration
Reads: 'sort-by' setting from node config/meta
Values: "database" (default) or "alphabetical"
```

##### `synchronize_data_types`
```
Purpose: Update column data types from database
Features:
  - Optional precision/scale for numerics
  - Optional length for strings
  - Respects output_to_lower setting
```

##### `synthesize_missing_documentation_with_openai`
```
Purpose: Generate AI documentation for undocumented columns/tables
Flow:
  1. First inherit any available upstream docs
  2. Identify undocumented columns
  3. If >10 undocumented: use bulk JSON generation
  4. Otherwise: individual column/table calls
Uses: llm.py module for API calls
```

#### Pipeline Construction Example

```python
# Create pipeline with >> operator
transform = (
    inject_missing_columns
    >> remove_columns_not_in_database
    >> inherit_upstream_column_knowledge
    >> sort_columns_as_configured
    >> synchronize_data_types
)

# Optionally add LLM synthesis
if synthesize:
    transform >>= synthesize_missing_documentation_with_openai

# Execute pipeline
transform(context=context)
```

---

### 3.4 inheritance.py - Documentation Inheritance

**Location**: `src/dbt_osmosis/core/inheritance.py` (200 lines)

#### Key Functions

##### `_build_node_ancestor_tree(manifest, node)`
```python
def _build_node_ancestor_tree(
    manifest: Manifest,
    node: ResultNode,
) -> dict[str, list[str]]:
    """
    Build a generational tree of ancestors.

    Returns:
        {
            "generation_0": ["current.node.id"],
            "generation_1": ["parent1.id", "parent2.id"],
            "generation_2": ["grandparent1.id"],
            ...
        }

    Uses depth-first traversal via depends_on.nodes.
    Only includes model., seed., source. nodes.
    """
```

##### `_get_node_yaml(context, member)`
```python
def _get_node_yaml(
    context: YamlRefactorContext,
    member: ResultNode
) -> MappingProxyType[str, Any] | None:
    """
    Get read-only view of parsed YAML for a node.

    For sources: Reads original_file_path, finds source→table
    For models/seeds: Reads patch_path, finds model section

    Returns None if no YAML documentation exists.
    """
```

##### `_build_column_knowledge_graph(context, node)`
```python
def _build_column_knowledge_graph(
    context: YamlRefactorContext,
    node: ResultNode
) -> dict[str, dict[str, Any]]:
    """
    Generate column documentation from all ancestors.

    Algorithm:
    1. Build ancestor tree
    2. Process ancestors from oldest to newest (reverse order)
    3. For each column in node:
       - Find matching column in ancestors (with fuzzy matching)
       - Merge description, tags, meta
       - Track progenitor if enabled

    Returns:
        {
            "column_name": {
                "description": "...",
                "tags": [...],
                "meta": {...},
                "osmosis_progenitor": "ancestor.node.id"
            }
        }
    """
```

#### Inheritance Rules

1. **Oldest-to-Newest**: Documentation from grandparents is overwritten by parents
2. **Tags Merge**: Union of all ancestor tags
3. **Meta Merge**: Dictionary merge, newer values win
4. **Description Priority**:
   - Skip placeholders (empty, "Not documented", etc.)
   - If `force_inherit_descriptions`: always use ancestor
   - Otherwise: only inherit if node column has no description
5. **Fuzzy Matching**: Plugins can provide column name variants (case, prefix)

---

### 3.5 introspection.py - Database Schema Discovery

**Location**: `src/dbt_osmosis/core/introspection.py` (311 lines)

#### Global Cache

```python
_COLUMN_LIST_CACHE: dict[str, OrderedDict[str, ColumnMetadata]] = {}
```
Caches column lists per relation to avoid redundant database queries.

#### Key Functions

##### `get_columns(context, relation)`
```python
def get_columns(
    context: YamlRefactorContext,
    relation: BaseRelation | ResultNode | None
) -> dict[str, ColumnMetadata]:
    """
    Get column metadata for a table/model.

    Priority:
    1. Check cache
    2. Check catalog.json if provided
    3. Query database via adapter.get_columns_in_relation()

    Returns OrderedDict preserving column order.
    Each ColumnMetadata contains: name, type, index, comment
    """
```

##### `normalize_column_name(column, credentials_type)`
```python
def normalize_column_name(column: str, credentials_type: str) -> str:
    """
    Normalize column names for comparison.

    - Snowflake: UPPERCASE (unless quoted)
    - Others: Strip quotes and backticks
    """
```

##### `_get_setting_for_node(opt, node, col, fallback)`
```python
def _get_setting_for_node(
    opt: str,
    node: ResultNode | None = None,
    col: str | None = None,
    fallback: Any = None
) -> Any:
    """
    Get configuration value with cascading lookup:

    1. Column meta: col.meta['dbt-osmosis-<opt>']
    2. Column meta: col.meta['dbt-osmosis-options']['<opt>']
    3. Node meta: node.meta['<opt>']
    4. Node meta: node.meta['dbt-osmosis-options']['<opt>']
    5. Node config: node.config.extra['dbt_osmosis_<opt>']
    6. Node config: node.config.extra['dbt_osmosis_options']['<opt>']
    7. Fallback value

    Supports both kebab-case and snake_case keys.
    """
```

##### `_load_catalog(settings)` / `_generate_catalog(context)`
```python
# Load existing catalog.json
def _load_catalog(settings) -> CatalogResults | None

# Generate new catalog via adapter introspection
def _generate_catalog(context) -> CatalogResults | None
```

---

### 3.6 restructuring.py - YAML File Reorganization

**Location**: `src/dbt_osmosis/core/restructuring.py` (344 lines)

#### Classes

##### `RestructureOperation`
```python
@dataclass
class RestructureOperation:
    file_path: Path                          # Target file path
    content: dict[str, Any]                  # YAML content to write
    superseded_paths: dict[Path, list[ResultNode]]  # Old files → nodes to remove
```

##### `RestructureDeltaPlan`
```python
@dataclass
class RestructureDeltaPlan:
    operations: list[RestructureOperation]   # All operations to perform
```

#### Key Functions

##### `draft_restructure_delta_plan(context)`
```python
def draft_restructure_delta_plan(context) -> RestructureDeltaPlan:
    """
    Analyze project and create migration plan.

    For each node where current_path != target_path:
    1. Create operation to write to target_path
    2. Track superseded_paths for cleanup

    Deduplicates operations targeting same file.
    Runs in parallel using context.pool.
    """
```

##### `apply_restructure_plan(context, plan, confirm)`
```python
def apply_restructure_plan(
    context: YamlRefactorContext,
    plan: RestructureDeltaPlan,
    confirm: bool = False
) -> None:
    """
    Execute the restructuring plan.

    1. If confirm=True: print plan and ask user
    2. For each operation:
       a. Read existing target file (if any)
       b. Merge operation content
       c. Write to target file
       d. Clean up superseded files:
          - Remove migrated models/sources/seeds
          - Delete empty YAML files
          - Remove empty directories
    3. Reload manifest to pick up changes
    """
```

##### Helper Functions

```python
def _generate_minimal_model_yaml(node: ModelNode | SeedNode) -> dict:
    """Create minimal YAML scaffold: {"name": ..., "columns": []}"""

def _generate_minimal_source_yaml(node: SourceDefinition) -> dict:
    """Create minimal source YAML: {"name": source_name, "tables": [...]}"""

def _remove_models(existing_doc, nodes) -> None:
    """Remove model entries from YAML doc"""

def _remove_seeds(existing_doc, nodes) -> None:
    """Remove seed entries from YAML doc"""

def _remove_sources(existing_doc, nodes) -> None:
    """Remove source/table entries from YAML doc"""
```

---

### 3.7 node_filters.py - Node Selection & Ordering

**Location**: `src/dbt_osmosis/core/node_filters.py` (151 lines)

#### Key Functions

##### `_is_fqn_match(node, fqns)`
```python
def _is_fqn_match(node: ResultNode, fqns: list[str]) -> bool:
    """
    Match node FQN against patterns.

    Example:
        node.fqn = ["project", "staging", "customers"]
        fqns = ["staging.customers"]  # Matches!
        fqns = ["staging"]            # Matches!
        fqns = ["marts"]              # No match
    """
```

##### `_is_file_match(node, paths, root)`
```python
def _is_file_match(node: ResultNode, paths: list[Path | str], root: Path) -> bool:
    """
    Match node against file/directory paths.

    Checks:
    - Model name matches path stem
    - Node path is under directory path
    - Node YAML path is under directory path
    - Exact file match
    """
```

##### `_topological_sort(candidate_nodes)`
```python
def _topological_sort(
    candidate_nodes: list[tuple[str, ResultNode]]
) -> list[tuple[str, ResultNode]]:
    """
    Sort nodes by dependency order using Kahn's algorithm.

    Ensures parents are processed before children.
    Critical for documentation inheritance to work correctly.

    Raises ValueError if cycle detected.
    """
```

##### `_iter_candidate_nodes(context, include_external)`
```python
def _iter_candidate_nodes(
    context: YamlRefactorContext,
    include_external: bool = False
) -> Iterator[tuple[str, ResultNode]]:
    """
    Iterate over filtered, sorted nodes.

    Filters:
    - Only Model, Source, Seed types
    - Only current project (unless include_external)
    - Skip ephemeral models
    - Apply FQN filter if set
    - Apply file path filter if set

    Returns topologically sorted results.
    """
```

---

### 3.8 path_management.py - YAML Path Resolution

**Location**: `src/dbt_osmosis/core/path_management.py` (250 lines)

#### Classes

##### `SchemaFileLocation`
```python
@dataclass
class SchemaFileLocation:
    target: Path           # Where the YAML should be
    current: Path | None   # Where the YAML currently is
    node_type: NodeType    # Model, Source, or Seed

    @property
    def is_valid(self) -> bool:
        """True if current == target (no migration needed)"""
```

##### `SchemaFileMigration`
```python
@dataclass
class SchemaFileMigration:
    output: dict[str, Any]                    # YAML content to write
    supersede: dict[Path, list[ResultNode]]   # Files to clean up
```

#### Key Functions

##### `_get_yaml_path_template(context, node)`
```python
def _get_yaml_path_template(context, node: ResultNode) -> str | None:
    """
    Get path template from configuration.

    For sources: Read from context.source_definitions[source_name]
    For models: Read from node config/meta:
        +dbt-osmosis: "{parent}/{node.name}.yml"

    Raises MissingOsmosisConfig if not configured.
    """
```

##### `get_target_yaml_path(context, node)`
```python
def get_target_yaml_path(context, node: ResultNode) -> Path:
    """
    Resolve target YAML path using template.

    Template variables:
    - {node}: The node object (access any attribute)
    - {model}: Model name
    - {parent}: Parent directory name
    - {node.fqn[0]}, {node.fqn[-1]}: FQN segments (supports negative index)
    - {node.tags[0]}, etc.: Tag segments

    Examples:
        "{parent}/_schema.yml"           → models/staging/_schema.yml
        "/{node.fqn[1]}/_schema.yml"     → models/staging/_schema.yml
        "_schema/{model}.yml"            → models/staging/_schema/customers.yml
    """
```

##### `build_yaml_file_mapping(context)`
```python
def build_yaml_file_mapping(
    context: YamlRefactorContext,
    create_missing_sources: bool = False
) -> dict[str, SchemaFileLocation]:
    """
    Build complete mapping of all nodes to their YAML locations.

    Returns dict: node_uid → SchemaFileLocation
    """
```

##### `create_missing_source_yamls(context)`
```python
def create_missing_source_yamls(context) -> None:
    """
    Bootstrap YAML files for sources defined in dbt_project.yml but not yet documented.

    Reads: vars.dbt-osmosis.sources configuration
    For each undefined source:
        1. Query database for tables in schema
        2. Get columns for each table
        3. Generate source YAML
        4. Write to specified path
        5. Reload manifest
    """
```

---

### 3.9 sync_operations.py - YAML Synchronization

**Location**: `src/dbt_osmosis/core/sync_operations.py` (252 lines)

#### Key Functions

##### `sync_node_to_yaml(context, node, commit)`
```python
def sync_node_to_yaml(
    context: YamlRefactorContext,
    node: ResultNode | None = None,
    commit: bool = True
) -> None:
    """
    Synchronize manifest node data to YAML file.

    One-way sync: Node → YAML

    For models/seeds:
        1. Find or create model entry in YAML
        2. Handle versioned models specially
        3. Sync columns, description, meta, tags

    For sources:
        1. Find or create source entry
        2. Find or create table entry
        3. Sync table columns and description

    Handles:
        - Duplicate model entries (deduplication)
        - Versioned models with versions array
        - Empty section cleanup
    """
```

##### `_sync_doc_section(context, node, doc_section)`
```python
def _sync_doc_section(
    context: YamlRefactorContext,
    node: ResultNode,
    doc_section: dict[str, Any]
) -> None:
    """
    Sync node data to a YAML section (model/table).

    For each column:
        1. Preserve existing YAML fields
        2. Merge node column data
        3. Apply skip_add_data_types setting
        4. Clean empty values
        5. Apply output_to_lower setting
    """
```

---

### 3.10 schema/ - YAML I/O Layer

**Location**: `src/dbt_osmosis/core/schema/`

#### parser.py
```python
def create_yaml_instance() -> ruamel.yaml.YAML:
    """
    Create configured YAML parser.

    Settings:
        - Indentation: mapping=2, sequence=4, offset=2
        - Width: 100 characters
        - Preserve quotes
        - Custom multiline string handling
    """
```

#### reader.py
```python
_YAML_BUFFER_CACHE: dict[Path, Any] = {}

def _read_yaml(
    yaml_handler: ruamel.yaml.YAML,
    yaml_handler_lock: threading.Lock,
    path: Path
) -> dict[str, Any]:
    """
    Read YAML file with caching.

    - Thread-safe via lock
    - Returns cached content if available
    - Returns empty dict for non-existent files
    """
```

#### writer.py
```python
def _write_yaml(
    yaml_handler, yaml_handler_lock, path, data, dry_run, mutation_tracker
) -> None:
    """
    Write YAML to disk.

    - Only writes if content changed
    - Creates parent directories
    - Clears cache entry
    - Tracks mutations
    """

def commit_yamls(
    yaml_handler, yaml_handler_lock, dry_run, mutation_tracker
) -> None:
    """
    Flush all cached YAML to disk.

    Used for batch commit mode in pipelines.
    """
```

---

## 4. CLI Commands Reference

### 4.1 Command Hierarchy

```
dbt-osmosis
├── yaml
│   ├── refactor    # Full refactoring (organize + document)
│   ├── organize    # Only restructure files
│   └── document    # Only documentation inheritance
├── sql
│   ├── run         # Execute SQL and show results
│   └── compile     # Compile Jinja SQL
├── workbench       # Start Streamlit UI
└── test_llm        # Test LLM provider connection
```

### 4.2 Common Options

| Option | Description |
|--------|-------------|
| `--project-dir` | Path to dbt_project.yml directory |
| `--profiles-dir` | Path to profiles.yml directory |
| `-t, --target` | dbt target to use |
| `--threads` | Number of parallel threads |
| `--log-level` | Logging level (default: INFO) |

### 4.3 yaml refactor

**Full YAML refactoring**: organizes files AND propagates documentation.

```bash
dbt-osmosis yaml refactor [OPTIONS] [MODELS]...
```

| Option | Description |
|--------|-------------|
| `-f, --fqn` | Filter by FQN (e.g., "staging.customers") |
| `-d, --dry-run` | Don't write changes to disk |
| `-C, --check` | Exit non-zero if changes made |
| `--catalog-path` | Use catalog.json instead of live queries |
| `--disable-introspection` | Run without database connection |
| `--auto-apply` | Skip restructure confirmation prompt |
| `-F, --force-inherit-descriptions` | Override existing descriptions |
| `--use-unrendered-descriptions` | Preserve `{{ doc(...) }}` blocks |
| `--skip-add-columns` | Don't add missing columns |
| `--skip-add-source-columns` | Don't add source columns |
| `--skip-add-tags` | Don't inherit tags |
| `--skip-merge-meta` | Don't merge meta fields |
| `--skip-add-data-types` | Don't add data types |
| `--add-progenitor-to-meta` | Track column origin |
| `--numeric-precision-and-scale` | Include numeric precision |
| `--string-length` | Include string lengths |
| `--output-to-lower` | Lowercase output |
| `--synthesize` | Enable LLM documentation |

### 4.4 yaml organize

**Only restructures** YAML files based on path rules.

```bash
dbt-osmosis yaml organize [OPTIONS] [MODELS]...
```

Same options as `refactor` except documentation-specific ones.

### 4.5 yaml document

**Only documentation inheritance**, no file restructuring.

```bash
dbt-osmosis yaml document [OPTIONS] [MODELS]...
```

Includes all documentation options but no `--auto-apply`.

### 4.6 sql run / sql compile

```bash
# Execute SQL and display results
dbt-osmosis sql run "SELECT * FROM {{ ref('customers') }} LIMIT 10"

# Just compile Jinja to SQL
dbt-osmosis sql compile "SELECT * FROM {{ ref('customers') }}"
```

### 4.7 workbench

```bash
dbt-osmosis workbench [OPTIONS]

Options:
  --host       Streamlit host (default: localhost)
  --port       Streamlit port (default: 8501)
  --options    Show streamlit run --help
  --config     Show streamlit config
```

### 4.8 test_llm

```bash
dbt-osmosis test_llm
```

Tests LLM provider connection using environment variables.

---

## 5. Data Flow Diagrams

### 5.1 refactor Command Flow

```
                                    User
                                      │
                                      ▼
                            ┌─────────────────┐
                            │ dbt-osmosis yaml│
                            │    refactor     │
                            └────────┬────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │ Create Configuration │
                          │ DbtConfiguration     │
                          └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │Create Project Context│
                          │ - Load manifest      │
                          │ - Init adapter       │
                          │ - Setup parsers      │
                          └──────────┬───────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │Create Refactor Ctx   │
                          │ - Settings           │
                          │ - Thread pool        │
                          │ - YAML handler       │
                          └──────────┬───────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    ▼                                  ▼
         ┌───────────────────┐              ┌────────────────────┐
         │create_missing_    │              │draft_restructure_  │
         │source_yamls       │              │delta_plan          │
         │                   │              │                    │
         │ For each source   │              │ Compare current vs │
         │ defined in vars:  │              │ target paths       │
         │ - Query tables    │              │                    │
         │ - Get columns     │              │ Generate migration │
         │ - Write YAML      │              │ operations         │
         └───────────────────┘              └──────────┬─────────┘
                                                       │
                                                       ▼
                                            ┌────────────────────┐
                                            │apply_restructure_  │
                                            │plan                │
                                            │                    │
                                            │ - Confirm with user│
                                            │ - Write new files  │
                                            │ - Clean old files  │
                                            │ - Reload manifest  │
                                            └──────────┬─────────┘
                                                       │
                                                       ▼
                                            ┌────────────────────┐
                                            │ Transform Pipeline │
                                            └──────────┬─────────┘
                                                       │
                    ┌──────────┬──────────┬───────────┬───────────┐
                    ▼          ▼          ▼           ▼           ▼
              ┌──────────┐┌─────────┐┌─────────┐┌─────────┐┌──────────┐
              │ inject   ││ remove  ││ inherit ││  sort   ││synchronize│
              │ missing  ││ extra   ││upstream ││ columns ││data types │
              │ columns  ││ columns ││knowledge││         ││           │
              └────┬─────┘└────┬────┘└────┬────┘└────┬────┘└─────┬────┘
                   │           │          │          │           │
                   └───────────┴──────────┴──────────┴───────────┘
                                          │
                                          ▼
                                ┌────────────────────┐
                                │ (Optional)         │
                                │ synthesize_with_llm│
                                └──────────┬─────────┘
                                           │
                                           ▼
                                ┌────────────────────┐
                                │ sync_node_to_yaml  │
                                │ (batch commit)     │
                                └──────────┬─────────┘
                                           │
                                           ▼
                                ┌────────────────────┐
                                │ Write YAML files   │
                                │ to disk            │
                                └────────────────────┘
```

### 5.2 Documentation Inheritance Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Source: raw_customers                            │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ columns:                                                          │  │
│  │   - name: customer_id                                             │  │
│  │     description: "Unique customer identifier"                     │  │
│  │   - name: email                                                   │  │
│  │     description: "Customer email address"                         │  │
│  │     tags: [pii]                                                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ depends_on
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Model: stg_customers                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ columns:                                                          │  │
│  │   - name: customer_id                                             │  │
│  │     description: ""  ← INHERITS "Unique customer identifier"      │  │
│  │   - name: email                                                   │  │
│  │     description: ""  ← INHERITS "Customer email address"          │  │
│  │     tags: []         ← INHERITS [pii]                             │  │
│  │   - name: created_at                                              │  │
│  │     description: ""  ← No upstream, stays empty or LLM generates  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ depends_on
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Model: dim_customers                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ columns:                                                          │  │
│  │   - name: customer_id                                             │  │
│  │     description: ""  ← INHERITS from stg_customers                │  │
│  │     meta:                                                         │  │
│  │       osmosis_progenitor: source.raw.raw_customers                │  │
│  │   - name: email                                                   │  │
│  │     description: ""  ← INHERITS from stg_customers                │  │
│  │     tags: [pii]      ← INHERITS from stg_customers                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

### 5.3 YAML Restructuring Flow

```
BEFORE                                    AFTER
======                                    =====

models/
├── staging/
│   ├── customers.sql
│   ├── orders.sql                        models/
│   └── _schema.yml  ◄─── Contains all    ├── staging/
│         models:                         │   ├── customers.sql
│           - name: customers             │   ├── customers.yml  ◄─── One file per model
│             columns: [...]              │   │     models:
│           - name: orders                │   │       - name: customers
│             columns: [...]              │   │         columns: [...]
│                                         │   ├── orders.sql
└── marts/                                │   └── orders.yml  ◄─── One file per model
    ├── core/                             │         models:
    │   └── dim_customers.sql             │           - name: orders
    └── schema.yml  ◄─── At wrong level   │             columns: [...]
          models:                         │
            - name: dim_customers         └── marts/
              columns: [...]                  └── core/
                                                  ├── dim_customers.sql
                                                  └── dim_customers.yml  ◄─── Moved to correct location
                                                        models:
                                                          - name: dim_customers
                                                            columns: [...]

Configuration in dbt_project.yml:
─────────────────────────────────
models:
  my_project:
    staging:
      +dbt-osmosis: "{model}.yml"     # One file per model
    marts:
      +dbt-osmosis: "{parent}/_schema.yml"  # One file per directory
```

---

## 6. Configuration Reference

### 6.1 dbt_project.yml Configuration

```yaml
name: my_project
version: '1.0.0'

vars:
  dbt-osmosis:                        # or dbt_osmosis (both work)
    # Source definitions for auto-bootstrapping
    sources:
      raw_customers:                  # Source name
        path: "sources/raw_customers.yml"  # Relative to model paths
        database: "raw_db"            # Optional: override database
        schema: "customers"           # Optional: override schema

      raw_orders: "sources/raw_orders.yml"  # Short form: just path

    # Column ignore patterns (regex)
    column_ignore_patterns:
      - "^_.*"                        # Skip columns starting with _
      - ".*_deprecated$"              # Skip deprecated columns

    # YAML handler settings
    yaml_settings:
      width: 120                      # Line width
      indent: 4                       # Indentation

models:
  my_project:
    # Path template for YAML files
    +dbt-osmosis: "_schema/{model}.yml"  # One file per model in _schema folder

    staging:
      +dbt-osmosis: "{model}.yml"     # One file per model, same directory

    marts:
      +dbt-osmosis: "{parent}/_schema.yml"  # One file per directory
```

### 6.2 Path Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{model}` | Model name | `customers` |
| `{parent}` | Parent directory name | `staging` |
| `{node}` | Full node object | Access any attribute |
| `{node.name}` | Node name | `customers` |
| `{node.fqn[0]}` | First FQN segment | `my_project` |
| `{node.fqn[-1]}` | Last FQN segment | `customers` |
| `{node.tags[0]}` | First tag | `core` |

### 6.3 Per-Model Configuration

In schema YAML or SQL config:

```yaml
# In schema YAML
models:
  - name: my_model
    meta:
      dbt-osmosis-options:
        skip-add-columns: true
        skip-add-data-types: true
        sort-by: alphabetical
        output-to-lower: true
        prefix: "src_"              # Strip prefix for column matching
    columns:
      - name: customer_id
        meta:
          dbt-osmosis-skip-merge-meta: true  # Per-column override
```

```sql
-- In SQL file
{{ config(
    dbt_osmosis_options={
        "skip_add_columns": true,
        "prefix": "account_"
    }
) }}
```

### 6.4 Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DBT_PROJECT_DIR` | Path to dbt_project.yml | CWD or parents |
| `DBT_PROFILES_DIR` | Path to profiles.yml | `~/.dbt` |
| `DBT_TARGET` | dbt target name | Profile default |
| `DBT_PROFILE` | dbt profile name | dbt_project.yml |
| `DBT_THREADS` | Number of threads | Profile default |
| **LLM Variables** | | |
| `LLM_PROVIDER` | LLM provider | `openai` |
| `OPENAI_API_KEY` | OpenAI API key | Required for OpenAI |
| `OPENAI_MODEL` | OpenAI model | `gpt-4o` |
| `AZURE_OPENAI_BASE_URL` | Azure endpoint | Required for Azure |
| `AZURE_OPENAI_API_KEY` | Azure API key | Required for Azure |
| `AZURE_OPENAI_API_VERSION` | Azure API version | `2025-01-01-preview` |
| `AZURE_OPENAI_DEPLOYMENT_NAME` | Azure deployment | Required for Azure |
| `GOOGLE_GEMINI_API_KEY` | Gemini API key | Required for Gemini |
| `GOOGLE_GEMINI_MODEL` | Gemini model | `gemini-2.0-flash` |
| `ANTHROPIC_API_KEY` | Anthropic API key | Required for Anthropic |
| `ANTHROPIC_MODEL` | Claude model | `claude-3-5-haiku-latest` |
| `OLLAMA_BASE_URL` | Ollama server URL | `http://localhost:11434/v1` |
| `OLLAMA_MODEL` | Ollama model | `llama2:latest` |
| `LM_STUDIO_BASE_URL` | LM Studio URL | `http://localhost:1234/v1` |
| `LM_STUDIO_MODEL` | LM Studio model | `local-model` |
| `OSMOSIS_LLM_MAX_SQL_CHARS` | Max SQL in prompts | Unlimited |

---

## 7. Plugin System

### 7.1 Architecture

dbt-osmosis uses **pluggy** for extensible column name matching:

```python
# Plugin specification
@_hookspec
def get_candidates(name: str, node: ResultNode, context: Any) -> list[str]:
    """Return alternative column names for matching."""
```

### 7.2 Built-in Plugins

#### FuzzyCaseMatching
```python
class FuzzyCaseMatching:
    def get_candidates(self, name, node, context) -> list[str]:
        # Returns: lowercase, UPPERCASE, camelCase, PascalCase
        # Example: "user_email" → ["user_email", "USER_EMAIL", "userEmail", "UserEmail"]
```

#### FuzzyPrefixMatching
```python
class FuzzyPrefixMatching:
    def get_candidates(self, name, node, context) -> list[str]:
        # Strips prefix defined in meta.prefix
        # Example: "src_customer_id" with prefix="src_" → ["customer_id"]
```

### 7.3 Creating Custom Plugins

```python
# my_plugin.py
from dbt_osmosis.core.plugins import hookimpl

class MyCustomMatching:
    @hookimpl
    def get_candidates(self, name: str, node, context) -> list[str]:
        # Custom logic
        return [name.replace("_", ""), name.upper()]

# Register in pyproject.toml or setup.py
[project.entry-points."dbt-osmosis"]
my_plugin = "my_plugin:MyCustomMatching"
```

---

## 8. LLM Integration

### 8.1 Supported Providers

| Provider | Environment Variables |
|----------|----------------------|
| OpenAI | `OPENAI_API_KEY`, `OPENAI_MODEL` |
| Azure OpenAI | `AZURE_OPENAI_*` (base_url, api_key, deployment_name, api_version) |
| Google Gemini | `GOOGLE_GEMINI_API_KEY`, `GOOGLE_GEMINI_MODEL` |
| Anthropic | `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` |
| Ollama | `OLLAMA_BASE_URL`, `OLLAMA_MODEL` |
| LM Studio | `LM_STUDIO_BASE_URL`, `LM_STUDIO_MODEL` |

### 8.2 Documentation Generation Modes

#### Individual Mode (< 10 undocumented columns)
```
For each undocumented column/table:
  1. Generate table description if missing
  2. Generate column description
Uses: generate_table_doc(), generate_column_doc()
```

#### Bulk Mode (>= 10 undocumented columns)
```
Single API call for entire model:
  1. Generate JSON with model description + all columns
Uses: generate_model_spec_as_json()
```

### 8.3 Context Building

The LLM receives:
1. Model SQL code (truncated if `OSMOSIS_LLM_MAX_SQL_CHARS` set)
2. Existing model description
3. Upstream model/column documentation (limited context)
4. Column data types

---

## 9. Workbench (Streamlit UI)

### 9.1 Features

- **SQL Editor**: Write and edit dbt SQL with Jinja support
- **Compilation**: See compiled SQL in real-time
- **Execution**: Run queries and view results
- **Model Browser**: Navigate dbt project models
- **Data Profiling**: Profile query results with ydata-profiling
- **RSS Feed**: Aggregated data news

### 9.2 Starting Workbench

```bash
dbt-osmosis workbench --host localhost --port 8501
```

### 9.3 Requirements

Requires `workbench` extra dependencies:
```bash
pip install "dbt-osmosis[workbench]"
```

---

## 10. Maintenance Guide

### 10.1 Regular Maintenance Tasks

#### Weekly Tasks

| Task | Description | Priority |
|------|-------------|----------|
| **Dependency Updates** | Check for dbt-core, ruamel.yaml updates | Medium |
| **Test Suite** | Run full test suite, fix failures | High |
| **Lint Check** | Run `ruff check src/` | Medium |

#### Monthly Tasks

| Task | Description | Priority |
|------|-------------|----------|
| **Security Audit** | Check dependencies for vulnerabilities | High |
| **Performance Review** | Profile slow operations | Low |
| **Documentation Sync** | Update docs with code changes | Medium |

#### Quarterly Tasks

| Task | Description | Priority |
|------|-------------|----------|
| **dbt-core Compatibility** | Test with new dbt releases | High |
| **Python Version Support** | Test with new Python versions | Medium |
| **Major Feature Planning** | Roadmap review | Low |

### 10.2 Critical Maintenance Areas

#### 1. dbt-core Compatibility (HIGH PRIORITY)

**Files to monitor:**
- `src/dbt_osmosis/core/config.py` - dbt API usage
- `src/dbt_osmosis/core/introspection.py` - Catalog/adapter APIs

**Common breaking changes:**
- Manifest structure changes
- Adapter interface changes
- RuntimeConfig API changes

**Testing:**
```bash
# Test with multiple dbt versions
pip install dbt-core==1.8.0 && pytest
pip install dbt-core==1.9.0 && pytest
pip install dbt-core==1.10.0 && pytest
```

#### 2. YAML Handling (MEDIUM PRIORITY)

**Files to monitor:**
- `src/dbt_osmosis/core/schema/parser.py`
- `src/dbt_osmosis/core/schema/reader.py`
- `src/dbt_osmosis/core/schema/writer.py`

**Potential issues:**
- ruamel.yaml version compatibility
- Round-trip preservation
- Comment handling

#### 3. LLM Integration (MEDIUM PRIORITY)

**Files to monitor:**
- `src/dbt_osmosis/core/llm.py`

**Potential issues:**
- OpenAI SDK version changes
- API endpoint changes
- Rate limiting

#### 4. Thread Safety (HIGH PRIORITY)

**Critical sections:**
- `_YAML_BUFFER_CACHE` access
- `_COLUMN_LIST_CACHE` access
- Manifest reloading
- Adapter connection management

**Review:**
- All uses of `yaml_handler_lock`
- All uses of `_manifest_mutex`
- Thread pool operations

### 10.3 Testing Strategy

#### Unit Tests
```bash
pytest tests/core/  # Core module tests
```

#### Integration Tests
```bash
pytest tests/test_yaml_*.py  # YAML integration tests
```

#### Manual Testing
```bash
# Test with sample project
cd sample_project/
dbt-osmosis yaml refactor --dry-run
dbt-osmosis yaml refactor --check
```

### 10.4 Common Issues & Solutions

#### Issue: "MissingOsmosisConfig"
**Cause**: Model missing `+dbt-osmosis` config
**Solution**: Add path template to dbt_project.yml

#### Issue: "Cycle detected in node dependencies"
**Cause**: Circular ref in dbt models
**Solution**: Fix model dependencies

#### Issue: "Could not introspect columns"
**Cause**: Database connection issue
**Solution**: Check profile credentials, use `--catalog-path`

#### Issue: "LLM returned empty response"
**Cause**: API error or rate limiting
**Solution**: Check API key, add retries

### 10.5 Performance Optimization

#### Caching
- Leverage `_YAML_BUFFER_CACHE` for repeated reads
- Use `_COLUMN_LIST_CACHE` for introspection
- Set appropriate `connection_ttl` for long runs

#### Parallelization
- Adjust `--threads` based on database capacity
- Use `--catalog-path` to avoid parallel DB queries

#### Memory
- Clear caches after large operations
- Use `--models` to limit scope

### 10.6 Upgrade Path

When upgrading dbt-osmosis:

1. **Read CHANGELOG** for breaking changes
2. **Backup YAML files** before running
3. **Test with `--dry-run`** first
4. **Review changes** before committing

---

## 11. Troubleshooting

### 11.1 Debug Mode

```bash
dbt-osmosis yaml refactor --log-level DEBUG
```

### 11.2 Common Error Messages

| Error | Cause | Solution |
|-------|-------|----------|
| `DBT_PROJECT_DIR not found` | Not in dbt project | cd to project or set env var |
| `Profile not found` | Invalid profile name | Check profiles.yml |
| `Adapter connection failed` | Database unreachable | Check credentials/network |
| `YAML parse error` | Invalid YAML syntax | Validate YAML files |
| `Node not found in manifest` | Model not compiled | Run `dbt parse` first |

### 11.3 Log Locations

- Console: Rich formatted output
- File: `~/.dbt-osmosis/logs/`

### 11.4 Reporting Issues

When reporting issues, include:
1. dbt-osmosis version: `pip show dbt-osmosis`
2. dbt-core version: `dbt --version`
3. Python version: `python --version`
4. Full error traceback
5. Relevant dbt_project.yml config
6. Minimal reproduction steps

---

## Appendix A: Function Reference Quick Index

| Module | Key Functions |
|--------|--------------|
| `config.py` | `create_dbt_project_context()`, `discover_project_dir()`, `_reload_manifest()` |
| `settings.py` | `YamlRefactorContext`, `YamlRefactorSettings` |
| `transforms.py` | `inject_missing_columns`, `inherit_upstream_column_knowledge`, `sort_columns_as_configured`, `synchronize_data_types`, `synthesize_missing_documentation_with_openai` |
| `inheritance.py` | `_build_column_knowledge_graph()`, `_build_node_ancestor_tree()` |
| `introspection.py` | `get_columns()`, `normalize_column_name()`, `_get_setting_for_node()` |
| `restructuring.py` | `draft_restructure_delta_plan()`, `apply_restructure_plan()` |
| `node_filters.py` | `_iter_candidate_nodes()`, `_topological_sort()` |
| `path_management.py` | `get_target_yaml_path()`, `build_yaml_file_mapping()`, `create_missing_source_yamls()` |
| `sync_operations.py` | `sync_node_to_yaml()` |
| `llm.py` | `generate_model_spec_as_json()`, `generate_column_doc()`, `generate_table_doc()` |

---

## Appendix B: Dependencies

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| click | 7.x - 8.x | CLI framework |
| dbt-core | 1.8 - 1.10.x | dbt integration |
| ruamel.yaml | 0.17 - 0.18.x | YAML parsing with round-trip |
| rich | 10+ | Console output |
| pluggy | 1.5+ | Plugin system |
| mysql-mimic | 2.5.7+ | SQL proxy server |

### Optional Dependencies

| Extra | Packages | Purpose |
|-------|----------|---------|
| `workbench` | streamlit, streamlit-ace, ydata-profiling | Streamlit UI |
| `openai` | openai | LLM documentation |
| `dev` | ruff, pytest, pre-commit, dbt-duckdb | Development |

---

*Generated for dbt-osmosis v1.1.17*
*Last Updated: December 2024*
