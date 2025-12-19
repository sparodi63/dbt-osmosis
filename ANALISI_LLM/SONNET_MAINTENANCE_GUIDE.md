# dbt-osmosis - Comprehensive Maintenance Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Detailed Function Documentation](#detailed-function-documentation)
3. [Architecture Diagrams](#architecture-diagrams)
4. [Maintenance Tasks Breakdown](#maintenance-tasks-breakdown)
5. [Upgrade Paths](#upgrade-paths)
6. [Known Issues and Workarounds](#known-issues-and-workarounds)
7. [Performance Tuning](#performance-tuning)

---

## Project Overview

**dbt-osmosis** is a CLI tool that automates YAML schema management for dbt (data build tool) projects. It was created to solve the repetitive and error-prone task of maintaining schema YAML files manually.

### Core Purpose
- **Automate** YAML file creation and organization
- **Propagate** documentation from upstream to downstream models
- **Synchronize** column definitions with database schema
- **Provide** an interactive workbench for SQL development

### Current State (v1.1.17)
- Python versions: 3.10, 3.11, 3.12, 3.13
- dbt-core: 1.8.x - 1.10.x
- Active development: **Paused** (no recent updates on main branch)
- Last significant update: Custom version constraint changes for dbt 1.10.x

---

## Detailed Function Documentation

### Core Modules

#### 1. **transforms.py** - Transformation Pipeline

##### `TransformOperation`
```python
@dataclass
class TransformOperation:
    func: t.Callable[..., t.Any]
    name: str
```

**Purpose**: Wraps a transformation function with metadata tracking and composition capabilities.

**Key Methods**:
- `__call__(context, node)`: Executes the transformation and tracks success/failure
- `__rshift__(next_op)`: Enables operation chaining using `>>` operator
- `result`: Returns the operation result
- `metadata`: Returns execution metadata (started, success, error, duration)

**Usage Example**:
```python
transform = inject_missing_columns >> remove_columns_not_in_database
result = transform(context=context, node=node)
```

---

##### `TransformPipeline`
```python
@dataclass
class TransformPipeline:
    operations: list[TransformOperation]
    commit_mode: Literal["none", "batch", "atomic", "defer"] = "batch"
```

**Purpose**: Manages a sequence of transformations with different commit strategies.

**Commit Modes**:
- `none`: No commits (changes stay in memory)
- `batch`: Commit all changes after pipeline completion
- `atomic`: Commit after each operation
- `defer`: Defer commits until program exit (using atexit)

**Key Features**:
- **Timing**: Tracks duration of each operation and total pipeline time
- **Logging**: Rich console output with progress indicators
- **Error Handling**: Captures errors in metadata without stopping pipeline

**Location**: `src/dbt_osmosis/core/transforms.py:78`

---

##### `inject_missing_columns(context, node)`
**Decorator**: `@_transform_op("Inject Missing Columns")`

**Purpose**: Adds columns from the database schema that are missing in the YAML file.

**Algorithm**:
```
1. Check if skip-add-columns is enabled → exit if true
2. Get current columns from node (YAML)
3. Get incoming columns from database (via introspection)
4. For each incoming column not in current:
   a. Create ColumnInfo with name, description, data_type
   b. Add to node.columns dictionary
```

**Settings Respected**:
- `skip-add-columns`: Skips entire operation
- `skip-add-source-columns`: Skips for source nodes only
- `skip-add-data-types`: Omits data_type from generated columns
- `output-to-lower`: Lowercases column names and types

**Performance**: Uses parallel processing via `context.pool.map()` when node is None

**Location**: `src/dbt_osmosis/core/transforms.py:242`

---

##### `remove_columns_not_in_database(context, node)`
**Decorator**: `@_transform_op("Remove Extra Columns")`

**Purpose**: Removes columns from YAML that no longer exist in the database.

**Algorithm**:
```
1. Get current columns from node (YAML) → normalize names
2. Get incoming columns from database
3. Calculate extra_columns = current - incoming
4. For each extra column:
   a. Log removal
   b. Pop from node.columns
```

**Safety Features**:
- Logs each removal for audit trail
- Uses normalized column names for comparison (case-insensitive on Snowflake/BigQuery/Redshift)

**Location**: `src/dbt_osmosis/core/transforms.py:290`

---

##### `inherit_upstream_column_knowledge(context, node)`
**Decorator**: `@_transform_op("Inherit Upstream Column Knowledge")`

**Purpose**: Propagates documentation, tags, and metadata from upstream models to downstream models.

**Algorithm**:
```
1. Build column knowledge graph for node (see inheritance.py)
2. For each column in node:
   a. Get inherited knowledge from graph
   b. Determine inheritable fields:
      - Always: "description"
      - Optional: "tags" (if not skip-add-tags)
      - Optional: "meta" (if not skip-merge-meta)
      - Custom: add-inheritance-for-specified-keys
   c. Update node.columns with inherited metadata
```

**Inheritance Priority**:
- Direct (generation_0) ancestors take precedence
- Earlier generations override later generations
- `force-inherit-descriptions` overrides existing empty descriptions

**Location**: `src/dbt_osmosis/core/transforms.py:184`

---

##### `sort_columns_as_configured(context, node)`
**Decorator**: `@_transform_op("Sort Columns")`

**Purpose**: Reorders columns based on configuration.

**Sort Methods**:
1. `database`: Orders columns as they appear in database schema (default)
2. `alphabetical`: Orders columns alphabetically by name

**Algorithm**:
```
1. Get sort-by setting for node (default: "database")
2. If "database":
   a. Get columns from database with index positions
   b. Sort node.columns by database position
3. If "alphabetical":
   a. Sort node.columns alphabetically by key
```

**Location**: `src/dbt_osmosis/core/transforms.py:376`

---

##### `synchronize_data_types(context, node)`
**Decorator**: `@_transform_op("Synchronize Data Types")`

**Purpose**: Updates data types in YAML to match database schema.

**Algorithm**:
```
1. Get incoming columns from database
2. For each column in node:
   a. Check if skip-add-data-types → skip if true
   b. Get incoming column metadata
   c. Update node.columns[name].data_type with incoming type
   d. Apply lowercase if output-to-lower is true
```

**Type Handling**:
- Preserves existing lowercase formatting if already lowercase
- Respects `output-to-lower` setting
- Handles precision/scale for numeric types (if enabled)
- Handles length for string types (if enabled)

**Location**: `src/dbt_osmosis/core/transforms.py:398`

---

##### `synthesize_missing_documentation_with_openai(context, node)`
**Decorator**: `@_transform_op("Synthesize Missing Documentation")`

**Purpose**: Uses OpenAI's GPT-4o to generate documentation for undocumented columns.

**Algorithm**:
```
1. Check if OpenAI extra is installed → raise ImportError if not
2. Inherit upstream knowledge first (reduces synthesis requests)
3. Count documented vs undocumented columns
4. Build upstream documentation context (limited to 100 entries)
5. If many undocumented columns (>10):
   a. Use bulk synthesis (one API call for entire model)
   b. Generate model spec with all columns
6. Else:
   a. Synthesize table description if missing
   b. Synthesize each column description individually
7. Update node.description and node.columns[].description
```

**Optimization**:
- Leverages inheritance to minimize API calls
- Bulk synthesis for tables with many undocumented columns
- Context window management (bounded to ~100 upstream docs)
- Temperature settings: 0.4 for bulk, 0.7 for individual

**Requirements**:
- `openai` package installed (`pip install "dbt-osmosis[openai]"`)
- Environment variable or setting with OpenAI API key

**Location**: `src/dbt_osmosis/core/transforms.py:435`

---

#### 2. **inheritance.py** - Documentation Inheritance

##### `_build_node_ancestor_tree(manifest, node, tree, visited, depth)`

**Purpose**: Builds a hierarchical tree of all ancestor nodes (models, seeds, sources) that a node depends on.

**Algorithm**:
```
1. Initialize:
   - tree = {"generation_0": [node.unique_id]}
   - visited = set(node.unique_id)
   - depth = 1
2. For each dependency in node.depends_on.nodes:
   a. Filter to model/seed/source nodes only
   b. Skip if already visited (prevents cycles)
   c. Add to tree[f"generation_{depth}"]
   d. Recursively build tree for dependency (depth + 1)
3. Sort each generation for deterministic ordering
4. Return tree
```

**Output Example**:
```python
{
  "generation_0": ["model.project.final_customers"],
  "generation_1": ["model.project.int_customers", "model.project.stg_orders"],
  "generation_2": ["source.salesforce.customers", "source.salesforce.orders"]
}
```

**Location**: `src/dbt_osmosis/core/inheritance.py:18`

---

##### `_get_node_yaml(context, member)`

**Purpose**: Retrieves the parsed YAML definition for a model, seed, or source node.

**Algorithm**:
```
1. Determine node type (SourceDefinition vs ModelNode/SeedNode)
2. If SourceDefinition:
   a. Read YAML from original_file_path
   b. Find source by source_name
   c. Find table within source
   d. Return MappingProxyType (read-only dict)
3. If ModelNode/SeedNode:
   a. Read YAML from patch_path
   b. Find model/seed by name in appropriate section
   c. Return MappingProxyType
4. Return None if not found
```

**Return Type**: `MappingProxyType[str, Any] | None`
- Read-only dict prevents accidental mutations
- Contains raw YAML data (columns, description, meta, tags, etc.)

**Location**: `src/dbt_osmosis/core/inheritance.py:51`

---

##### `_build_column_knowledge_graph(context, node)`

**Purpose**: Creates a comprehensive knowledge graph showing what documentation each column should inherit.

**Algorithm**:
```
1. Build ancestor tree for node
2. Initialize column variants (for fuzzy matching)
   a. For each column, get plugin candidates
   b. Store variants (e.g., "CustomerId" → ["CustomerId", "customer_id", "CUSTOMER_ID"])
3. Initialize empty knowledge graph: dict[column_name, dict[field, value]]
4. Iterate ancestors in reverse order (oldest to newest):
   a. For each ancestor:
      i. For each column in node:
         - Find matching column in ancestor (using variants)
         - Extract: description, tags, meta, custom keys
         - Remove empty tags/meta if flags set
         - Add progenitor to meta if configured
         - Use unrendered descriptions if configured
         - Merge tags and meta with existing knowledge
         - Remove placeholders and None values
         - Update knowledge graph
5. Return knowledge graph
```

**Key Features**:
- **Fuzzy Matching**: Uses plugin system for flexible column matching
- **Generation Awareness**: Closer ancestors override distant ones
- **Merge Strategy**: Tags are union, meta is deep merge
- **Placeholder Handling**: Removes placeholder descriptions (empty strings, "TODO", etc.)
- **Progenitor Tracking**: Can track which model originated a column's documentation

**Output Example**:
```python
{
  "customer_id": {
    "description": "Unique identifier for customer",
    "tags": ["pii", "primary_key"],
    "meta": {"osmosis_progenitor": "source.salesforce.customers"}
  },
  "email": {
    "description": "Customer email address",
    "tags": ["pii"]
  }
}
```

**Location**: `src/dbt_osmosis/core/inheritance.py:90`

---

#### 3. **cli/main.py** - Command-Line Interface

##### Main Commands

**`cli()`**
- Entry point for all dbt-osmosis commands
- Groups: `yaml`, `sql`, `workbench`
- Version option displays current package version

**Location**: `src/dbt_osmosis/cli/main.py:46`

---

##### `yaml refactor` Command

**Purpose**: Complete workflow combining organization and documentation.

**Execution Flow**:
```
1. Parse CLI arguments
2. Create DbtConfiguration and YamlRefactorContext
3. Create missing source YAMLs
4. Draft restructure plan
5. Apply restructure plan (with confirmation unless --auto-apply)
6. Execute transformation pipeline:
   - inject_missing_columns
   - remove_columns_not_in_database
   - inherit_upstream_column_knowledge
   - sort_columns_as_configured
   - synchronize_data_types
   - (optional) synthesize_missing_documentation_with_openai
7. Exit with code 1 if --check and mutations detected
```

**Key Options**:
- `--force-inherit-descriptions`: Override existing descriptions
- `--skip-add-columns`: Don't add missing columns
- `--skip-add-tags`: Don't inherit tags
- `--skip-merge-meta`: Don't merge meta fields
- `--numeric-precision-and-scale`: Include precision/scale in types
- `--auto-apply`: Skip confirmation prompt
- `--synthesize`: Use OpenAI for missing documentation
- `--dry-run`: Preview changes without committing
- `--check`: Exit with error if changes detected (for CI/CD)

**Location**: `src/dbt_osmosis/cli/main.py:282`

---

##### `yaml organize` Command

**Purpose**: Restructure YAML files based on configuration without changing documentation.

**Execution Flow**:
```
1. Parse CLI arguments
2. Create DbtConfiguration and YamlRefactorContext
3. Create missing source YAMLs
4. Draft restructure plan
5. Apply restructure plan (with confirmation unless --auto-apply)
6. Exit with code 1 if --check and mutations detected
```

**Use Case**: Initial setup or reorganization of YAML files without documentation changes.

**Location**: `src/dbt_osmosis/cli/main.py:350`

---

##### `yaml document` Command

**Purpose**: Inherit documentation without reorganizing YAML files.

**Execution Flow**:
```
1. Parse CLI arguments
2. Create DbtConfiguration and YamlRefactorContext
3. Execute transformation pipeline:
   - inject_missing_columns
   - inherit_upstream_column_knowledge
   - sort_columns_as_configured
   - (optional) synthesize_missing_documentation_with_openai
4. Exit with code 1 if --check and mutations detected
```

**Use Case**: Update documentation in existing YAML files without changing file structure.

**Location**: `src/dbt_osmosis/cli/main.py:466`

---

##### `sql run` Command

**Purpose**: Execute dbt SQL statements with Jinja rendering and output results.

**Execution Flow**:
```
1. Parse SQL statement and connection parameters
2. Create DbtProjectContext
3. Execute SQL via execute_sql_code(project, sql)
4. Format and print results to stdout
```

**Output Format**: Table with max 50 rows, 6 columns, 20 char column width

**Location**: `src/dbt_osmosis/cli/main.py:605`

---

##### `sql compile` Command

**Purpose**: Compile dbt SQL statements without executing them.

**Execution Flow**:
```
1. Parse SQL statement and connection parameters
2. Create DbtProjectContext
3. Compile SQL via compile_sql_code(project, sql)
4. Print compiled SQL to stdout
```

**Use Case**: Debug Jinja templates, inspect compiled SQL before execution.

**Location**: `src/dbt_osmosis/cli/main.py:636`

---

##### `workbench` Command

**Purpose**: Launch Streamlit-based interactive workbench.

**Execution Flow**:
```
1. Parse CLI arguments
2. Handle special flags:
   - --options: Show streamlit options
   - --config: Show streamlit config
3. Build streamlit command with:
   - Custom host and port
   - Magic disabled (runner.magicEnabled=false)
   - Script path (workbench/app.py)
   - Pass-through arguments
4. Execute streamlit via subprocess
5. Exit with streamlit's return code
```

**Default**: localhost:8501

**Location**: `src/dbt_osmosis/cli/main.py:546`

---

## Architecture Diagrams

### 1. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        dbt-osmosis CLI                          │
│                      (cli/main.py)                              │
└────────────┬────────────────────────┬───────────────────────────┘
             │                        │
             │                        │
┌────────────▼──────────┐  ┌──────────▼─────────────────────────┐
│   YAML Management     │  │      SQL Operations                │
│   (yaml commands)     │  │      (sql commands)                │
│                       │  │                                    │
│  - refactor           │  │  - run (execute)                   │
│  - organize           │  │  - compile                         │
│  - document           │  │  - workbench (Streamlit)           │
└────────────┬──────────┘  └──────────┬─────────────────────────┘
             │                        │
             │                        │
┌────────────▼────────────────────────▼─────────────────────────┐
│                    Core Services                               │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Configuration│  │  Inheritance │  │ Introspection│         │
│  │  (config.py) │  │(inheritance  │  │(introspection│         │
│  │              │  │     .py)     │  │     .py)     │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Transforms  │  │ Restructuring│  │   Plugins    │         │
│  │(transforms   │  │(restructuring│  │  (plugins.py)│         │
│  │    .py)      │  │     .py)     │  │              │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Schema I/O   │  │ Path Mgmt    │  │     LLM      │         │
│  │(schema/*.py) │  │(path_mgmt.py)│  │   (llm.py)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└────────────┬────────────────────────────────────────┬──────────┘
             │                                        │
┌────────────▼──────────┐                  ┌──────────▼─────────┐
│    dbt-core           │                  │    Database        │
│    - Manifest         │                  │    - Schema        │
│    - Compilation      │                  │    - Metadata      │
│    - SQL Runner       │                  │    - Catalog       │
└───────────────────────┘                  └────────────────────┘
```

---

### 2. Transformation Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│              yaml refactor / document Command               │
└────────────┬────────────────────────────────────────────────┘
             │
             │ Create Context
             │
┌────────────▼─────────────────────────────────────────────────┐
│                  YamlRefactorContext                         │
│  - DbtProjectContext (manifest, profiles, runtime_cfg)       │
│  - YamlRefactorSettings (all flags and options)              │
│  - Pool (multiprocessing)                                    │
│  - YAML Handler (ruamel.yaml)                                │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Build Pipeline
             │
┌────────────▼─────────────────────────────────────────────────┐
│                  TransformPipeline                           │
│                                                              │
│  inject_missing_columns                                      │
│          ↓                                                   │
│  remove_columns_not_in_database                              │
│          ↓                                                   │
│  inherit_upstream_column_knowledge                           │
│          ↓                                                   │
│  sort_columns_as_configured                                  │
│          ↓                                                   │
│  synchronize_data_types                                      │
│          ↓                                                   │
│  (optional) synthesize_missing_documentation_with_openai     │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Execute Pipeline
             │
┌────────────▼─────────────────────────────────────────────────┐
│            For Each Node (Model/Seed/Source)                 │
│                                                              │
│  1. Get columns from database (introspection)                │
│  2. Get columns from YAML (current state)                    │
│  3. Apply transformation:                                    │
│     - Add missing columns                                    │
│     - Remove extra columns                                   │
│     - Inherit documentation                                  │
│     - Sort columns                                           │
│     - Sync data types                                        │
│  4. Buffer changes in memory                                 │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Commit Changes
             │
┌────────────▼─────────────────────────────────────────────────┐
│                  commit_yamls()                              │
│                                                              │
│  1. For each modified YAML file:                             │
│     - Read current YAML                                      │
│     - Merge buffered changes                                 │
│     - Write updated YAML                                     │
│  2. Track mutations for --check flag                         │
└──────────────────────────────────────────────────────────────┘
```

---

### 3. Documentation Inheritance Flow

```
┌──────────────────────────────────────────────────────────────┐
│         inherit_upstream_column_knowledge(context, node)     │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 1: Build Ancestor Tree
             │
┌────────────▼─────────────────────────────────────────────────┐
│         _build_node_ancestor_tree(manifest, node)            │
│                                                              │
│  Input: node = "model.proj.final_customers"                  │
│  Output:                                                     │
│    generation_0: ["model.proj.final_customers"]              │
│    generation_1: ["model.proj.int_customers"]                │
│    generation_2: ["model.proj.stg_customers"]                │
│    generation_3: ["source.salesforce.customers"]             │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 2: Build Knowledge Graph
             │
┌────────────▼─────────────────────────────────────────────────┐
│        _build_column_knowledge_graph(context, node)          │
│                                                              │
│  For each ancestor (from oldest to newest):                  │
│    1. Get YAML definition (_get_node_yaml)                   │
│    2. For each column in node:                               │
│       a. Find matching column in ancestor (fuzzy)            │
│       b. Extract: description, tags, meta                    │
│       c. Merge with existing knowledge                       │
│       d. Track progenitor if configured                      │
│                                                              │
│  Output: column_knowledge_graph                              │
│    {                                                         │
│      "customer_id": {                                        │
│        "description": "Unique ID",                           │
│        "tags": ["pii"],                                      │
│        "meta": {"progenitor": "source..."}                   │
│      },                                                      │
│      ...                                                     │
│    }                                                         │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 3: Apply Knowledge
             │
┌────────────▼─────────────────────────────────────────────────┐
│               Update node.columns                            │
│                                                              │
│  For each column in node:                                    │
│    1. Get inherited knowledge from graph                     │
│    2. Filter by settings (skip-add-tags, skip-merge-meta)    │
│    3. Update column with: description, tags, meta            │
│    4. Remove empty values (tags=[], meta={})                 │
└──────────────────────────────────────────────────────────────┘
```

---

### 4. Restructuring Flow

```
┌──────────────────────────────────────────────────────────────┐
│                 yaml organize / refactor                     │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 1: Create Missing Source YAMLs
             │
┌────────────▼─────────────────────────────────────────────────┐
│          create_missing_source_yamls(context)                │
│                                                              │
│  - Reads vars.dbt-osmosis.sources config                     │
│  - Creates YAML files for unconfigured sources               │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 2: Draft Restructure Plan
             │
┌────────────▼─────────────────────────────────────────────────┐
│          draft_restructure_delta_plan(context)               │
│                                                              │
│  For each model/seed:                                        │
│    1. Get current YAML path (if exists)                      │
│    2. Get target YAML path (from config)                     │
│    3. Determine operation:                                   │
│       - CREATE: No current YAML                              │
│       - MOVE: Different paths, single model in YAML          │
│       - MERGE: Different paths, multiple models in YAML      │
│       - DELETE: YAML file no longer needed                   │
│                                                              │
│  Output: RestructureDeltaPlan                                │
│    {                                                         │
│      "create": [RestructureOperation(...)],                  │
│      "move": [RestructureOperation(...)],                    │
│      "merge": [RestructureOperation(...)],                   │
│      "delete": [RestructureOperation(...)]                   │
│    }                                                         │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 3: Display Plan
             │
┌────────────▼─────────────────────────────────────────────────┐
│              pretty_print_plan(plan)                         │
│                                                              │
│  Creates: staging/customers/_schema.yml                      │
│  Moves:   models/customers.yml → staging/customers/customers.yml│
│  Merges:  models/schema.yml → staging/schema.yml             │
│  Deletes: models/old_schema.yml                              │
└────────────┬─────────────────────────────────────────────────┘
             │
             │ Step 4: Confirm (unless --auto-apply)
             │
┌────────────▼─────────────────────────────────────────────────┐
│          apply_restructure_plan(context, plan, confirm)      │
│                                                              │
│  If confirm and not auto_apply:                              │
│    - Prompt user: "Apply these changes? [Y/n]"               │
│    - Exit if declined                                        │
│                                                              │
│  Execute operations in order:                                │
│    1. DELETE (remove old files)                              │
│    2. MOVE (relocate existing YAMLs)                         │
│    3. MERGE (combine multiple YAMLs)                         │
│    4. CREATE (bootstrap new YAMLs)                           │
│                                                              │
│  Track mutations for --check flag                            │
└──────────────────────────────────────────────────────────────┘
```

---

### 5. Configuration Hierarchy

```
┌──────────────────────────────────────────────────────────────┐
│                      Configuration Hierarchy                  │
│                   (Lower number = Higher priority)            │
└──────────────────────────────────────────────────────────────┘

Priority 1: Column-Level (Highest)
┌──────────────────────────────────────────────────────────────┐
│  YAML File: models/staging/customers.yml                     │
│                                                              │
│  columns:                                                    │
│    - name: customer_id                                       │
│      meta:                                                   │
│        dbt-osmosis-skip-add-data-types: true                 │
│        dbt-osmosis-sort-by: "alphabetical"                   │
└──────────────────────────────────────────────────────────────┘

Priority 2: Node-Level
┌──────────────────────────────────────────────────────────────┐
│  SQL File: models/staging/stg_customers.sql                  │
│                                                              │
│  {{ config(                                                  │
│      materialized='table',                                   │
│      dbt_osmosis_options={                                   │
│        "skip-add-columns": true,                             │
│        "sort-by": "alphabetical"                             │
│      }                                                       │
│  ) }}                                                        │
└──────────────────────────────────────────────────────────────┘

Priority 3: Folder-Level
┌──────────────────────────────────────────────────────────────┐
│  dbt_project.yml                                             │
│                                                              │
│  models:                                                     │
│    my_project:                                               │
│      staging:                                                │
│        +dbt-osmosis: "{parent}.yml"                          │
│        +dbt-osmosis-options:                                 │
│          skip-add-columns: true                              │
│          sort-by: "alphabetical"                             │
└──────────────────────────────────────────────────────────────┘

Priority 4: Global CLI Flags (Lowest)
┌──────────────────────────────────────────────────────────────┐
│  Command Line                                                │
│                                                              │
│  dbt-osmosis yaml refactor \                                 │
│    --skip-add-columns \                                      │
│    --numeric-precision-and-scale \                           │
│    --output-to-lower                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## Maintenance Tasks Breakdown

### Critical Maintenance Tasks

#### 1. **Dependency Updates**
**Priority**: HIGH
**Frequency**: Quarterly or when security vulnerabilities discovered

**Current Dependencies**:
```toml
click>=7,<9
dbt-core>=1.8,<1.11
ruamel.yaml>=0.17,<0.19
rich>=10
pluggy>=1.5.0,<2
mysql-mimic>=2.5.7
```

**Task Checklist**:
- [ ] Update `dbt-core` to latest compatible version
  - Test with all supported adapters (Snowflake, BigQuery, Redshift, Postgres, DuckDB)
  - Verify manifest structure hasn't changed
  - Check for breaking changes in dbt API
- [ ] Update `ruamel.yaml` for YAML parsing improvements
  - Test YAML reading/writing with edge cases
  - Verify comment preservation
- [ ] Update `click` for CLI improvements
  - Test all commands and options
  - Verify help text formatting
- [ ] Update `rich` for console output enhancements
  - Test logging output
  - Verify progress indicators
- [ ] Update `pluggy` for plugin system
  - Test custom plugin registration
  - Verify hook execution
- [ ] Update `streamlit` (workbench extra)
  - Test workbench UI
  - Verify compatibility with streamlit-ace

**Testing Commands**:
```bash
# Install dev dependencies
pip install -e ".[dev,workbench,openai]"

# Run test suite
pytest tests/

# Test specific adapters
dbt-osmosis yaml refactor --target=snowflake --dry-run
dbt-osmosis yaml refactor --target=bigquery --dry-run
dbt-osmosis yaml refactor --target=postgres --dry-run
```

**Files to Review**:
- `pyproject.toml:24-33` (dependencies)
- `pyproject.toml:35-54` (optional dependencies)

---

#### 2. **Python Version Support**
**Priority**: HIGH
**Frequency**: Annually or when new Python version released

**Current Support**: Python 3.10, 3.11, 3.12, 3.13

**Task Checklist**:
- [ ] Test with Python 3.13 (recently added support)
  - Review `tests/test_python_313_compatibility.py`
  - Run full test suite
  - Check for deprecation warnings
- [ ] Prepare for Python 3.14 (future)
  - Monitor Python 3.14 beta releases
  - Test with beta versions
  - Update type hints if needed
- [ ] Drop Python 3.10 support (future consideration)
  - Evaluate usage statistics
  - Announce deprecation 6 months ahead
  - Update classifiers in pyproject.toml

**Testing Matrix**:
```yaml
# .github/workflows/test.yml (create if doesn't exist)
strategy:
  matrix:
    python-version: ["3.10", "3.11", "3.12", "3.13"]
    dbt-version: ["1.8", "1.9", "1.10"]
```

**Files to Update**:
- `pyproject.toml:18-21` (classifiers)
- `pyproject.toml:23` (requires-python)
- GitHub Actions workflows

---

#### 3. **dbt-core Version Compatibility**
**Priority**: CRITICAL
**Frequency**: Every dbt-core release (quarterly)

**Current Support**: dbt-core 1.8.x - 1.10.x

**Task Checklist**:
- [ ] Monitor dbt-core releases
  - Subscribe to dbt-labs/dbt-core GitHub releases
  - Join dbt Slack #dbt-core-development channel
- [ ] Test new dbt-core versions
  - Create test project with new dbt-core version
  - Run all dbt-osmosis commands
  - Check for API changes
- [ ] Update version constraints
  - Test lower bound (currently 1.8)
  - Test upper bound (currently <1.11)
  - Update pyproject.toml
- [ ] Handle breaking changes
  - Review dbt-core CHANGELOG
  - Identify breaking API changes
  - Update affected code
  - Add compatibility shims if needed

**Known Issues**:
- Recent update (issue #278): Fixed problem with empty tags and meta
  - Location: `src/dbt_osmosis/core/inheritance.py:207-212`
  - Solution: Filter out empty tags and meta fields

**Files to Monitor**:
- `src/dbt_osmosis/core/config.py` (dbt project loading)
- `src/dbt_osmosis/core/sql_operations.py` (dbt compilation)
- `src/dbt_osmosis/core/introspection.py` (database introspection)

**Testing Commands**:
```bash
# Test with specific dbt version
pip install "dbt-core==1.10.0" "dbt-snowflake==1.10.0"
dbt-osmosis yaml refactor --dry-run

# Test manifest parsing
python -c "from dbt_osmosis.core.config import create_dbt_project_context; print('OK')"
```

---

#### 4. **Bug Fixes and Issue Triage**
**Priority**: HIGH
**Frequency**: Weekly

**Current Known Issues**:
- Issue #278: Empty tags and meta causing problems ✅ FIXED
- Issue #285: Loosen dbt-core version constraints ✅ RESOLVED
- Issue #270: Fix typing issues ✅ RESOLVED

**Task Checklist**:
- [ ] Review GitHub issues weekly
  - Triage new issues
  - Label appropriately (bug, enhancement, question)
  - Assign priority
- [ ] Reproduce reported bugs
  - Create minimal reproduction
  - Add test case
  - Document expected vs actual behavior
- [ ] Fix high-priority bugs
  - Write failing test first
  - Implement fix
  - Verify test passes
  - Update documentation if needed
- [ ] Update CHANGELOG
  - Document all changes
  - Follow semantic versioning
  - Prepare release notes

**Bug Fix Workflow**:
```bash
# 1. Create branch
git checkout -b fix/issue-XXX

# 2. Write test
# Add test to tests/ directory

# 3. Run test (should fail)
pytest tests/test_issue_XXX.py -v

# 4. Implement fix
# Modify source code

# 5. Run test (should pass)
pytest tests/test_issue_XXX.py -v

# 6. Run full test suite
pytest tests/ -v

# 7. Commit and push
git add .
git commit -m "fix: resolve issue #XXX"
git push origin fix/issue-XXX
```

---

#### 5. **Documentation Maintenance**
**Priority**: MEDIUM
**Frequency**: Monthly

**Current Documentation**:
- README.md (basic overview)
- DBT_OSMOSIS_SUMMARY.md (executive summary)
- DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md (detailed features)
- DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md (technical architecture)
- DBT_OSMOSIS_DOCUMENTATION_INDEX.md (navigation)
- DBT_OSMOSIS_MAINTENANCE_GUIDE.md (this document)
- Official docs site: https://z3z1ma.github.io/dbt-osmosis/

**Task Checklist**:
- [ ] Update documentation for new features
  - Add usage examples
  - Update configuration reference
  - Add screenshots/diagrams
- [ ] Fix documentation errors
  - Correct typos
  - Update outdated examples
  - Fix broken links
- [ ] Improve documentation clarity
  - Simplify complex explanations
  - Add more examples
  - Include troubleshooting tips
- [ ] Keep docs site in sync
  - Update official docs site
  - Ensure consistency across all docs
  - Version documentation

**Files to Maintain**:
- `README.md` (update version badges, examples)
- `docs/` directory (if exists)
- Documentation markdown files
- Inline code comments and docstrings

---

#### 6. **Performance Optimization**
**Priority**: MEDIUM
**Frequency**: Bi-annually

**Current Performance Features**:
- Parallel processing via `multiprocessing.Pool`
- Column metadata caching (`_COLUMN_LIST_CACHE`)
- YAML buffer caching (`_YAML_BUFFER_CACHE`)
- Batch YAML commits

**Task Checklist**:
- [ ] Profile performance bottlenecks
  - Use `cProfile` or `py-spy`
  - Identify slow operations
  - Measure before/after changes
- [ ] Optimize database queries
  - Minimize introspection queries
  - Batch column metadata retrieval
  - Use catalog.json when possible
- [ ] Optimize YAML I/O
  - Reduce YAML file reads
  - Batch YAML writes
  - Improve caching strategy
- [ ] Optimize transformation pipeline
  - Reduce redundant operations
  - Improve parallel processing
  - Optimize memory usage

**Profiling Commands**:
```bash
# Profile with cProfile
python -m cProfile -o profile.stats -m dbt_osmosis yaml refactor --dry-run

# Analyze profile
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative'); p.print_stats(20)"

# Profile with py-spy
py-spy record -o profile.svg -- dbt-osmosis yaml refactor --dry-run
```

**Performance Targets**:
- Small projects (<100 models): <30 seconds
- Medium projects (100-500 models): <2 minutes
- Large projects (500-1000 models): <10 minutes
- Very large projects (>1000 models): <30 minutes

**Files to Optimize**:
- `src/dbt_osmosis/core/transforms.py` (transformation operations)
- `src/dbt_osmosis/core/introspection.py` (database queries)
- `src/dbt_osmosis/core/schema/reader.py` (YAML reading)
- `src/dbt_osmosis/core/schema/writer.py` (YAML writing)

---

#### 7. **Testing and Quality Assurance**
**Priority**: HIGH
**Frequency**: Continuous

**Current Test Coverage**: Unknown (needs measurement)

**Task Checklist**:
- [ ] Measure test coverage
  - Install `pytest-cov`
  - Run coverage analysis
  - Identify untested code paths
- [ ] Increase test coverage to >80%
  - Write unit tests for core functions
  - Write integration tests for commands
  - Write end-to-end tests for workflows
- [ ] Add regression tests for fixed bugs
  - Each bug fix should include test
  - Test edge cases
  - Test error handling
- [ ] Set up CI/CD pipeline
  - Run tests on every commit
  - Test multiple Python versions
  - Test multiple dbt-core versions
  - Test multiple adapters

**Testing Commands**:
```bash
# Install test dependencies
pip install -e ".[dev]"

# Run tests
pytest tests/ -v

# Run tests with coverage
pytest tests/ --cov=src/dbt_osmosis --cov-report=html

# Run specific test
pytest tests/test_transforms.py::test_inject_missing_columns -v

# Run tests for specific Python version
tox -e py310
tox -e py311
tox -e py312
```

**Test Categories**:
1. **Unit Tests** (fast, isolated)
   - Test individual functions
   - Mock external dependencies
   - Test edge cases and error handling

2. **Integration Tests** (medium speed)
   - Test interaction between modules
   - Test with real dbt project
   - Test with test database

3. **End-to-End Tests** (slow, comprehensive)
   - Test full workflows
   - Test with real-world scenarios
   - Test different adapters

**Files to Test**:
- Priority 1: `transforms.py`, `inheritance.py`, `introspection.py`
- Priority 2: `restructuring.py`, `config.py`, `cli/main.py`
- Priority 3: `plugins.py`, `llm.py`, `workbench/app.py`

---

#### 8. **Security Audits**
**Priority**: HIGH
**Frequency**: Quarterly

**Task Checklist**:
- [ ] Scan for dependency vulnerabilities
  - Use `pip-audit` or `safety`
  - Review security advisories
  - Update vulnerable dependencies
- [ ] Review code for security issues
  - SQL injection prevention (already using parameterized queries)
  - Path traversal prevention (validate file paths)
  - Credential handling (ensure no credentials in logs)
  - OpenAI API key security (environment variables only)
- [ ] Review access controls
  - Ensure proper file permissions
  - Validate user inputs
  - Sanitize SQL and file paths
- [ ] Update security documentation
  - Document security best practices
  - Add security section to README
  - Provide secure configuration examples

**Security Scan Commands**:
```bash
# Install security tools
pip install pip-audit safety

# Run pip-audit
pip-audit

# Run safety check
safety check

# Check for outdated packages
pip list --outdated
```

**Security Considerations**:
1. **Database Credentials**
   - Never log credentials
   - Use environment variables or profiles.yml
   - Support credential rotation

2. **File System Access**
   - Validate all file paths
   - Prevent directory traversal
   - Respect file permissions

3. **SQL Injection**
   - Use parameterized queries
   - Validate column names
   - Escape special characters

4. **OpenAI API**
   - Validate API key format
   - Don't log API keys
   - Handle API errors gracefully

**Files to Review**:
- `src/dbt_osmosis/core/sql_operations.py` (SQL execution)
- `src/dbt_osmosis/core/introspection.py` (database queries)
- `src/dbt_osmosis/core/schema/reader.py` (file reading)
- `src/dbt_osmosis/core/schema/writer.py` (file writing)
- `src/dbt_osmosis/core/llm.py` (API key handling)

---

#### 9. **Code Quality and Linting**
**Priority**: MEDIUM
**Frequency**: Continuous

**Current Tools**:
- `ruff` (linting and formatting)
- `black` (code formatting - deprecated in favor of ruff)
- `isort` (import sorting - deprecated in favor of ruff)

**Task Checklist**:
- [ ] Run linters on every commit
  - Configure pre-commit hooks
  - Run `ruff check`
  - Fix all warnings
- [ ] Maintain consistent code style
  - Use `ruff format`
  - Follow PEP 8 guidelines
  - Use type hints
- [ ] Improve type coverage
  - Add type hints to all functions
  - Use `mypy` or `pyright` for type checking
  - Fix all type errors
- [ ] Reduce code complexity
  - Refactor complex functions
  - Extract reusable components
  - Improve readability

**Linting Commands**:
```bash
# Run ruff check
ruff check src/ tests/

# Run ruff format
ruff format src/ tests/

# Run ruff with auto-fix
ruff check --fix src/ tests/

# Run pyright (type checking)
pyright src/

# Run all quality checks
ruff check src/ tests/ && ruff format src/ tests/ && pyright src/
```

**Code Quality Metrics**:
- Cyclomatic complexity: <10 per function
- Line length: 100 characters (configured)
- Import order: sorted and organized
- Type coverage: >80%

**Configuration Files**:
- `pyproject.toml:73-75` (ruff config)
- `pyproject.toml:61-64` (black config - deprecated)
- `pyproject.toml:66-71` (isort config - deprecated)

---

#### 10. **Workbench Maintenance**
**Priority**: LOW
**Frequency**: As needed

**Current State**: Streamlit-based workbench for interactive SQL development

**Task Checklist**:
- [ ] Update Streamlit dependencies
  - Monitor Streamlit releases
  - Test UI compatibility
  - Update streamlit-ace if needed
- [ ] Fix workbench bugs
  - Test all features
  - Fix broken components
  - Improve error handling
- [ ] Add new workbench features
  - Data profiling enhancements
  - Query history
  - Saved queries
  - Dark mode improvements
- [ ] Optimize workbench performance
  - Reduce load times
  - Improve query execution
  - Cache compiled SQL

**Testing Commands**:
```bash
# Start workbench locally
dbt-osmosis workbench --project-dir=./test-project

# Test with specific port
dbt-osmosis workbench --port=8080

# View streamlit options
dbt-osmosis workbench --options
```

**Files to Maintain**:
- `src/dbt_osmosis/workbench/app.py` (main app)
- `src/dbt_osmosis/workbench/components/*.py` (UI components)

---

### Regular Maintenance Schedule

#### Weekly Tasks
- [ ] Review new GitHub issues
- [ ] Triage and label issues
- [ ] Respond to community questions
- [ ] Monitor CI/CD pipeline
- [ ] Review pull requests

#### Monthly Tasks
- [ ] Update documentation
- [ ] Review and close stale issues
- [ ] Update CHANGELOG
- [ ] Check for outdated dependencies
- [ ] Review security advisories

#### Quarterly Tasks
- [ ] Update dependencies
- [ ] Run security audit
- [ ] Performance profiling
- [ ] Review test coverage
- [ ] Plan next release

#### Annual Tasks
- [ ] Python version support review
- [ ] Major version planning
- [ ] Architecture review
- [ ] Roadmap update
- [ ] Community survey

---

## Upgrade Paths

### Upgrading Dependencies

#### dbt-core Upgrade (1.10 → 1.11)
```bash
# 1. Create test environment
python -m venv venv-test
source venv-test/bin/activate

# 2. Install new dbt-core
pip install "dbt-core==1.11.0" "dbt-snowflake==1.11.0"

# 3. Install dbt-osmosis in dev mode
pip install -e ".[dev]"

# 4. Run tests
pytest tests/ -v

# 5. Test with real project
dbt-osmosis yaml refactor --project-dir=/path/to/project --dry-run

# 6. If successful, update pyproject.toml
# Change: dbt-core>=1.8,<1.11
# To:     dbt-core>=1.8,<1.12
```

#### Python Version Upgrade (3.13 → 3.14)
```bash
# 1. Install Python 3.14
pyenv install 3.14.0

# 2. Create test environment
pyenv virtualenv 3.14.0 osmosis-3.14
pyenv activate osmosis-3.14

# 3. Install dependencies
pip install -e ".[dev,workbench,openai]"

# 4. Run tests
pytest tests/ -v

# 5. Check for deprecation warnings
python -W all -m pytest tests/

# 6. Update pyproject.toml if successful
# Add "Programming Language :: Python :: 3.14"
# Change requires-python to ">=3.10,<3.15"
```

---

### Upgrading dbt-osmosis

#### For Users

**Minor Version Upgrade (1.1.x → 1.2.x)**
```bash
# 1. Backup your dbt project
cp -r /path/to/dbt-project /path/to/dbt-project-backup

# 2. Upgrade dbt-osmosis
pip install --upgrade dbt-osmosis

# 3. Test with dry-run
dbt-osmosis yaml refactor --dry-run

# 4. Review changes
# Check that configuration is still valid
# Verify no breaking changes

# 5. Apply changes
dbt-osmosis yaml refactor
```

**Major Version Upgrade (1.x → 2.x)**
```bash
# 1. Read migration guide
# Check: https://z3z1ma.github.io/dbt-osmosis/docs/migrating

# 2. Review CHANGELOG
# Identify breaking changes
# Plan configuration updates

# 3. Test in separate environment
python -m venv venv-test
source venv-test/bin/activate
pip install "dbt-osmosis>=2.0"

# 4. Update configuration
# Modify dbt_project.yml as needed
# Update CLI flags

# 5. Test migration
dbt-osmosis yaml refactor --dry-run

# 6. Apply in production
pip install --upgrade "dbt-osmosis>=2.0"
dbt-osmosis yaml refactor
```

---

## Known Issues and Workarounds

### Issue: Empty Tags and Meta Fields (Fixed in recent commit)

**Symptom**: YAML files contain empty `tags: []` or `meta: {}` fields

**Root Cause**: Inheritance system wasn't filtering out empty collections

**Fix**: Applied in commit `3476a5a` - filters empty tags and meta
- Location: `src/dbt_osmosis/core/inheritance.py:207-212`
- Location: `src/dbt_osmosis/core/transforms.py:232-235`

**Workaround** (if using older version):
```yaml
# Manual cleanup in dbt_project.yml
models:
  your_project:
    +dbt-osmosis-options:
      skip-add-tags: true
      skip-merge-meta: true
```

---

### Issue: Performance with Large Projects

**Symptom**: Slow execution on projects with >1000 models

**Root Cause**: Sequential processing, repeated database queries

**Workaround**:
```bash
# 1. Use catalog.json to avoid live queries
dbt docs generate
dbt-osmosis yaml refactor --catalog-path target/catalog.json

# 2. Process in batches using FQN filter
dbt-osmosis yaml refactor --fqn staging
dbt-osmosis yaml refactor --fqn intermediate
dbt-osmosis yaml refactor --fqn marts

# 3. Use --disable-introspection for offline work
dbt-osmosis yaml refactor --disable-introspection --catalog-path target/catalog.json
```

**Future Fix**: Implement better parallelization and caching

---

### Issue: Compatibility with Specific dbt Adapters

**Symptom**: Errors with less common dbt adapters (Databricks, Trino, etc.)

**Root Cause**: Adapter-specific column introspection logic

**Workaround**:
```bash
# Use catalog.json for unsupported adapters
dbt docs generate
dbt-osmosis yaml refactor --catalog-path target/catalog.json
```

**Contributing**: Add adapter support in `src/dbt_osmosis/core/introspection.py`

---

### Issue: OpenAI Rate Limits

**Symptom**: Rate limit errors when using `--synthesize`

**Root Cause**: Too many API calls for large projects

**Workaround**:
```bash
# 1. Process in smaller batches
dbt-osmosis yaml document --fqn staging --synthesize
sleep 60
dbt-osmosis yaml document --fqn marts --synthesize

# 2. Use inheritance first to minimize synthesis
dbt-osmosis yaml document  # Without --synthesize
dbt-osmosis yaml document --synthesize  # Only for remaining

# 3. Increase rate limit with OpenAI
# Contact OpenAI to increase your rate limit
```

---

## Performance Tuning

### Optimization Techniques

#### 1. Use Catalog for Introspection
```bash
# Generate catalog once
dbt docs generate

# Use catalog for all subsequent runs
dbt-osmosis yaml refactor --catalog-path target/catalog.json
```

**Performance Gain**: 50-80% faster (no database queries)

---

#### 2. Process in Parallel (Already Implemented)
The codebase already uses `multiprocessing.Pool` for parallel processing:

```python
# In transforms.py
for _ in context.pool.map(
    partial(inject_missing_columns, context),
    (n for _, n in _iter_candidate_nodes(context)),
):
    ...
```

**Configuration**: Set `--threads` to match CPU cores
```bash
dbt-osmosis yaml refactor --threads=8
```

---

#### 3. Batch Processing by FQN
```bash
# Process specific folders
dbt-osmosis yaml refactor --fqn staging
dbt-osmosis yaml refactor --fqn intermediate
dbt-osmosis yaml refactor --fqn marts
```

**Use Case**: Incremental updates, targeted refactoring

---

#### 4. Disable Unnecessary Operations
```bash
# Skip operations you don't need
dbt-osmosis yaml refactor \
  --skip-add-columns \
  --skip-add-data-types \
  --skip-add-tags
```

**Performance Gain**: 20-40% faster

---

#### 5. Use Dry-Run for Testing
```bash
# Test changes without committing
dbt-osmosis yaml refactor --dry-run
```

**Performance Gain**: Faster feedback loop

---

### Profiling and Benchmarking

#### Profiling Command
```bash
# Profile with cProfile
python -m cProfile -s cumulative -m dbt_osmosis yaml refactor --dry-run 2>&1 | head -50
```

#### Benchmark Script
```bash
#!/bin/bash
# benchmark.sh

echo "Benchmarking dbt-osmosis..."

# Warmup
dbt-osmosis yaml refactor --dry-run > /dev/null 2>&1

# Benchmark with catalog
echo "With catalog:"
time dbt-osmosis yaml refactor --catalog-path target/catalog.json --dry-run

# Benchmark without catalog
echo "Without catalog:"
time dbt-osmosis yaml refactor --dry-run

# Benchmark with --skip-add-columns
echo "With --skip-add-columns:"
time dbt-osmosis yaml refactor --skip-add-columns --dry-run
```

---

## Conclusion

This maintenance guide provides comprehensive information for maintaining and improving dbt-osmosis. Key priorities:

1. **Keep dependencies updated** (especially dbt-core)
2. **Monitor and fix bugs** reported by community
3. **Maintain test coverage** above 80%
4. **Document all changes** in CHANGELOG
5. **Performance optimization** for large projects

### Next Steps for Maintainers

1. Set up automated CI/CD pipeline
2. Increase test coverage to >80%
3. Create contribution guidelines (CONTRIBUTING.md)
4. Establish regular release cadence
5. Improve documentation with more examples
6. Enhance performance for large projects (>1000 models)
7. Add support for more dbt adapters
8. Consider major version 2.0 with breaking changes for cleanup

### Community Engagement

- **GitHub**: https://github.com/z3z1ma/dbt-osmosis
- **Documentation**: https://z3z1ma.github.io/dbt-osmosis/
- **Issues**: https://github.com/z3z1ma/dbt-osmosis/issues
- **Discussions**: https://github.com/z3z1ma/dbt-osmosis/discussions

---

**Last Updated**: 2024-12-18
**Version**: 1.1.17
**Maintainer**: Community (original author: z3z1ma)
