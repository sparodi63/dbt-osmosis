# dbt-osmosis: Comprehensive Technical Documentation

**Version:** 1.1.17  
**Original Author:** Alex Butler (z3z1ma)  
**License:** Apache 2.0  
**Python Support:** 3.10 - 3.12  
**dbt-core Compatibility:** 1.8.x - 1.10.x

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [Module Reference](#module-reference)
5. [Configuration Reference](#configuration-reference)
6. [CLI Commands](#cli-commands)
7. [Data Flow Diagrams](#data-flow-diagrams)
8. [Plugin System](#plugin-system)
9. [Maintenance Tasks](#maintenance-tasks)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Executive Summary

dbt-osmosis is a developer productivity tool that enhances the dbt (data build tool) development experience through automated YAML schema management, column-level documentation inheritance, and an interactive development workbench. The project addresses common pain points in dbt development, particularly the tedious manual maintenance of schema YAML files.

### Core Value Propositions

1. **DRY Documentation**: Define column documentation once at the source level; propagate it automatically to all downstream models
2. **Automated Schema Management**: Generate, organize, and synchronize YAML schema files with database schemas
3. **Interactive Development**: Streamlit-powered workbench for real-time SQL compilation and query testing
4. **LLM-Powered Documentation**: Optional OpenAI integration to auto-generate missing documentation
5. **CI/CD Integration**: Pre-commit hooks and `--check` flags for pipeline integration

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           dbt-osmosis                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐    ┌─────────────────┐    ┌───────────────────┐    │
│  │   CLI Layer    │    │  Workbench UI   │    │   Python API      │    │
│  │  (click-based) │    │  (Streamlit)    │    │  (Programmatic)   │    │
│  └───────┬────────┘    └────────┬────────┘    └─────────┬─────────┘    │
│          │                      │                       │               │
│          └──────────────────────┼───────────────────────┘               │
│                                 │                                        │
│                    ┌────────────▼────────────┐                          │
│                    │   Core Osmosis Module   │                          │
│                    │    (osmosis.py)         │                          │
│                    └────────────┬────────────┘                          │
│                                 │                                        │
│   ┌─────────────────────────────┼─────────────────────────────────┐     │
│   │                             │                                  │     │
│   ▼                             ▼                                  ▼     │
│ ┌──────────────┐     ┌──────────────────┐           ┌──────────────┐    │
│ │   YAML       │     │  Transformation  │           │   LLM        │    │
│ │  Management  │     │    Pipeline      │           │  Module      │    │
│ └──────────────┘     └──────────────────┘           └──────────────┘    │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                        External Dependencies                             │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │  dbt-core  │  │ ruamel.yaml  │  │   pluggy    │  │    OpenAI     │  │
│  └────────────┘  └──────────────┘  └─────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### Package Structure

```
dbt_osmosis/
├── __init__.py                 # Package initialization
├── __main__.py                 # Entry point (calls cli.main)
├── cli/
│   ├── __init__.py
│   └── main.py                 # Click CLI implementation (~600 lines)
├── core/
│   ├── __init__.py
│   ├── osmosis.py              # Core functionality (~2500 lines)
│   ├── logger.py               # Rich logging configuration
│   └── llm.py                  # OpenAI integration (~330 lines)
├── sql/
│   ├── __init__.py
│   └── proxy.py                # MySQL proxy server (mysql-mimic)
└── workbench/
    ├── __init__.py
    ├── app.py                  # Streamlit application
    ├── requirements.txt
    └── components/             # UI components
```

---

## Core Components

### 1. DbtConfiguration

**Location:** `core/osmosis.py` (Lines 139-158)

A dataclass that encapsulates all configuration needed to initialize a dbt project context.

```python
@dataclass
class DbtConfiguration:
    project_dir: str              # Path to dbt project
    profiles_dir: str             # Path to profiles.yml directory
    target: str | None            # Target profile to use
    profile: str | None           # Profile name override
    threads: int | None           # Parallelism threads
    single_threaded: bool | None  # Force single-threaded mode
    vars: dict[str, Any]          # Runtime variables
    quiet: bool = True            # Suppress dbt output
    disable_introspection: bool   # Skip warehouse queries
```

### 2. DbtProjectContext

**Location:** `core/osmosis.py` (Lines 179-248)

The central context object that holds references to the dbt runtime configuration, manifest, parsers, and database adapter. Implements lazy adapter initialization with TTL-based connection recycling.

**Key Properties:**
- `config`: The DbtConfiguration instance
- `runtime_cfg`: dbt RuntimeConfig object
- `manifest`: Parsed dbt Manifest
- `sql_parser`: SqlBlockParser for SQL compilation
- `macro_parser`: SqlMacroParser for macro compilation
- `adapter`: Lazy-loaded database adapter with mutex protection

### 3. YamlRefactorSettings

**Location:** `core/osmosis.py` (Lines 428-464)

Configuration for YAML refactoring operations with granular control over behavior.

```python
@dataclass
class YamlRefactorSettings:
    fqn: list[str]                           # FQN filter patterns
    models: list[Path | str]                 # File path filters
    dry_run: bool                            # Preview mode
    skip_merge_meta: bool                    # Skip meta inheritance
    skip_add_columns: bool                   # Skip column injection
    skip_add_tags: bool                      # Skip tag inheritance
    skip_add_data_types: bool                # Skip data type sync
    skip_add_source_columns: bool            # Skip source column injection
    add_progenitor_to_meta: bool             # Track column origins
    numeric_precision_and_scale: bool        # Include precision
    string_length: bool                      # Include string length
    force_inherit_descriptions: bool         # Override existing docs
    use_unrendered_descriptions: bool        # Preserve doc() refs
    add_inheritance_for_specified_keys: list # Custom inheritable keys
    output_to_lower: bool                    # Lowercase output
    catalog_path: str | None                 # Pre-generated catalog
    create_catalog_if_not_exists: bool       # Auto-generate catalog
```

### 4. YamlRefactorContext

**Location:** `core/osmosis.py` (Lines 467-567)

The execution context for YAML refactoring operations, combining project context with refactoring settings and runtime state.

**Key Features:**
- Thread pool executor (synced with dbt thread count)
- YAML handler (ruamel.yaml instance)
- Mutation tracking
- Catalog caching
- Placeholder detection

### 5. Transform Operations

**Location:** `core/osmosis.py` (Lines 1953-2100)

A functional pipeline system for chaining transformation operations on dbt nodes.

**Available Operations:**
| Operation | Description |
|-----------|-------------|
| `inject_missing_columns` | Add columns present in DB but missing from YAML |
| `remove_columns_not_in_database` | Remove columns not in DB |
| `inherit_upstream_column_knowledge` | Propagate docs from ancestors |
| `sort_columns_as_in_database` | Order columns by DB position |
| `sort_columns_alphabetically` | Alphabetical column ordering |
| `sort_columns_as_configured` | Use configured sort method |
| `synchronize_data_types` | Sync data types with DB |
| `synthesize_missing_documentation_with_openai` | LLM doc generation |

**Pipeline Example:**
```python
pipeline = (
    inject_missing_columns
    >> remove_columns_not_in_database
    >> inherit_upstream_column_knowledge
    >> sort_columns_as_configured
    >> synchronize_data_types
)
pipeline.commit_mode = "atomic"  # or "batch", "defer", "none"
pipeline(context=yaml_context)
```

---

## Module Reference

### core/osmosis.py

The central module containing all core functionality (~2500 lines).

#### Project Discovery Functions

| Function | Description |
|----------|-------------|
| `discover_project_dir()` | Find dbt_project.yml, checking DBT_PROJECT_DIR first |
| `discover_profiles_dir()` | Find profiles.yml, checking DBT_PROFILES_DIR first |

#### Context Factory Functions

| Function | Description |
|----------|-------------|
| `create_dbt_project_context(config)` | Build full DbtProjectContext |
| `create_yaml_instance(...)` | Create configured ruamel.yaml instance |

#### SQL Compilation & Execution

| Function | Description |
|----------|-------------|
| `compile_sql_code(context, raw_sql)` | Compile Jinja SQL to pure SQL |
| `execute_sql_code(context, raw_sql)` | Execute SQL and return results |

#### YAML Management

| Function | Description |
|----------|-------------|
| `create_missing_source_yamls(context)` | Bootstrap source YAMLs from DB |
| `get_current_yaml_path(context, node)` | Get node's current YAML path |
| `get_target_yaml_path(context, node)` | Calculate target YAML path |
| `build_yaml_file_mapping(context)` | Map all nodes to YAML locations |
| `draft_restructure_delta_plan(context)` | Plan file reorganization |
| `apply_restructure_plan(context, plan, confirm)` | Execute reorganization |
| `sync_node_to_yaml(context, node, commit)` | Write node state to YAML |
| `commit_yamls(context)` | Flush all pending YAML changes |

#### Column Introspection

| Function | Description |
|----------|-------------|
| `normalize_column_name(column, credentials_type)` | Normalize case per adapter |
| `get_columns(context, relation)` | Get columns from DB/catalog |

#### Knowledge Graph

| Function | Description |
|----------|-------------|
| `_build_node_ancestor_tree(manifest, node)` | Build dependency tree |
| `_build_column_knowledge_graph(context, node)` | Merge ancestor docs |

### core/llm.py

OpenAI integration for automated documentation generation (~330 lines).

| Function | Description |
|----------|-------------|
| `generate_model_spec_as_json(sql_content, ...)` | Generate full model JSON spec |
| `generate_column_doc(column_name, ...)` | Generate single column description |
| `generate_table_doc(sql_content, table_name, ...)` | Generate table description |

**Environment Variables:**
- `OPENAI_API_KEY`: Required for LLM features
- `OSMOSIS_LLM_MAX_SQL_CHARS`: Truncate SQL to avoid token limits

### core/logger.py

Rich console logging with emoji support for better visual feedback.

### cli/main.py

Click-based command-line interface implementation.

---

## Configuration Reference

### dbt_project.yml Configuration

The `+dbt-osmosis` directive controls YAML file organization:

```yaml
models:
  your_project:
    # Default: One YAML per model, prefixed with underscore
    +dbt-osmosis: "_{model}.yml"
    
    staging:
      # Group by parent folder name
      +dbt-osmosis: "{parent}.yml"
      
    intermediate:
      # Organize by materialization
      +dbt-osmosis: "{node.config[materialized]}/{model}.yml"
      
    marts:
      # Single file for all models
      +dbt-osmosis: "prod.yml"

seeds:
  your_project:
    +dbt-osmosis: "_schema.yml"
```

### Path Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{model}` | Model name | `customers` |
| `{parent}` | Parent folder name | `staging` |
| `{node.config[key]}` | Config value | `incremental` |
| `{node.tags[n]}` | Tag by index | `marketing` |
| `{fqn[n]}` | FQN segment (supports negative) | `marts` |

### Source Configuration

Configure source management in `dbt_project.yml`:

```yaml
vars:
  dbt-osmosis:
    sources:
      # Simple: source name → yaml path
      salesforce: "staging/salesforce/source.yml"
      
      # Complex: with schema override
      marketo:
        path: "staging/customer/marketo.yml"
        schema: "raw_marketo"
        database: "analytics_db"
    
    # Skip columns matching these patterns
    column_ignore_patterns:
      - "^_.*"           # Underscore-prefixed
      - ".*_TIMESTAMP$"  # Timestamp suffixes
    
    # ruamel.yaml settings
    yaml_settings:
      width: 120
      indent: 2
```

### Folder-Level Options

```yaml
models:
  your_project:
    staging:
      +dbt-osmosis: "{parent}.yml"
      +dbt-osmosis-options:
        skip-add-columns: true
        sort-by: "alphabetical"
        output-to-lower: true
```

### Node-Level Options (in SQL file)

```sql
{{ config(
    materialized='incremental',
    dbt_osmosis_options={
        "skip-add-data-types": True,
        "prefix": "stg_"
    }
) }}
```

### Column-Level Options (in schema.yml)

```yaml
models:
  - name: my_model
    columns:
      - name: special_column
        meta:
          dbt-osmosis-skip-add-data-types: true
          dbt-osmosis-options:
            output-to-lower: true
```

---

## CLI Commands

### Global Options

```bash
dbt-osmosis --version          # Show version
dbt-osmosis --help             # Show help
```

### yaml refactor

The main command that organizes files AND documents columns.

```bash
dbt-osmosis yaml refactor \
  --project-dir /path/to/project \
  --profiles-dir /path/to/profiles \
  --target prod \
  --threads 4 \
  --auto-apply \
  --force-inherit-descriptions \
  --use-unrendered-descriptions \
  --skip-add-columns \
  --skip-add-data-types \
  --skip-merge-meta \
  --skip-add-tags \
  --numeric-precision-and-scale \
  --string-length \
  --output-to-lower \
  --add-progenitor-to-meta \
  --catalog-path target/catalog.json \
  --dry-run \
  --check \
  --synthesize \
  models/staging/
```

### yaml organize

Reorganizes YAML files without modifying documentation.

```bash
dbt-osmosis yaml organize \
  --project-dir . \
  --auto-apply \
  --check
```

### yaml document

Updates documentation without reorganizing files.

```bash
dbt-osmosis yaml document \
  --project-dir . \
  --force-inherit-descriptions \
  --synthesize \
  -f staging.salesforce
```

### sql run

Execute SQL with Jinja compilation.

```bash
dbt-osmosis sql run "SELECT * FROM {{ ref('my_model') }} LIMIT 10"
```

### sql compile

Compile Jinja SQL without execution.

```bash
dbt-osmosis sql compile "SELECT {{ var('my_var') }}"
```

### workbench

Start the Streamlit development workbench.

```bash
dbt-osmosis workbench \
  --project-dir . \
  --profiles-dir ~/.dbt \
  --host localhost \
  --port 8501
```

---

## Data Flow Diagrams

### YAML Refactor Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        yaml refactor Command                             │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Create DbtProjectContext                                             │
│     • Load dbt_project.yml and profiles.yml                             │
│     • Parse manifest (compile all models)                                │
│     • Initialize database adapter                                        │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  2. Create Missing Source YAMLs                                          │
│     • Iterate sources defined in vars.dbt-osmosis.sources                │
│     • Query database for table schemas                                   │
│     • Generate source YAML files                                         │
│     • Reload manifest if sources were created                            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  3. Draft Restructure Plan                                               │
│     • For each model/seed/source in manifest:                            │
│       - Calculate target YAML path from +dbt-osmosis template            │
│       - Compare with current YAML path (patch_path)                      │
│       - Generate RestructureOperation if paths differ                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  4. Apply Restructure Plan                                               │
│     • Prompt for confirmation (unless --auto-apply)                      │
│     • For each operation:                                                │
│       - Read existing YAML content                                       │
│       - Extract node definition                                          │
│       - Merge into target file                                           │
│       - Remove from source file                                          │
│       - Delete empty source files                                        │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  5. Execute Transform Pipeline                                           │
│     For each node (topologically sorted):                                │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  inject_missing_columns                                         │ │
│     │  • Query database/catalog for columns                           │ │
│     │  • Add columns present in DB but missing from YAML              │ │
│     └────────────────────────────────┬────────────────────────────────┘ │
│                                      ▼                                   │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  remove_columns_not_in_database                                 │ │
│     │  • Remove columns from YAML not found in DB                     │ │
│     └────────────────────────────────┬────────────────────────────────┘ │
│                                      ▼                                   │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  inherit_upstream_column_knowledge                              │ │
│     │  • Build ancestor tree via depends_on_nodes                     │ │
│     │  • Build column knowledge graph                                 │ │
│     │  • Merge descriptions, tags, meta from ancestors                │ │
│     └────────────────────────────────┬────────────────────────────────┘ │
│                                      ▼                                   │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  sort_columns_as_configured                                     │ │
│     │  • Sort by database order OR alphabetically                     │ │
│     └────────────────────────────────┬────────────────────────────────┘ │
│                                      ▼                                   │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  synchronize_data_types                                         │ │
│     │  • Update data_type field from database schema                  │ │
│     └────────────────────────────────┬────────────────────────────────┘ │
│                                      ▼                                   │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  (Optional) synthesize_missing_documentation_with_openai        │ │
│     │  • For undocumented columns, call OpenAI API                    │ │
│     │  • Bulk generation for >10 missing columns                      │ │
│     └─────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  6. Commit YAMLs                                                         │
│     • Sync all modified nodes to their YAML files                       │
│     • Preserve formatting with ruamel.yaml                              │
│     • Report mutation count                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Column Knowledge Inheritance Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Column Knowledge Inheritance                         │
└────────────────────────────────────────────────────────────────────────┘

Source: raw_salesforce.contacts
├── contact_id: "Unique identifier for the contact" [tags: PII, GDPR]
├── email: "Contact's email address" [tags: PII]
└── created_at: "Record creation timestamp"
         │
         │ ref('stg_salesforce__contacts')
         ▼
Staging: stg_salesforce__contacts
├── contact_id: → INHERITED: "Unique identifier for the contact" [tags: PII, GDPR]
├── email: → INHERITED: "Contact's email address" [tags: PII]  
├── created_at: → INHERITED: "Record creation timestamp"
└── full_name: "Contact's full name" [NEW - added in staging]
         │
         │ ref('int_contacts_enriched')
         ▼
Intermediate: int_contacts_enriched
├── contact_id: → INHERITED (via stg → source)
├── email: → INHERITED (via stg → source)
├── created_at: → INHERITED (via stg → source)
├── full_name: → INHERITED (via stg)
└── enrichment_score: "Score from enrichment API" [NEW]
         │
         │ ref('dim_contacts')
         ▼
Mart: dim_contacts
├── contact_id: → INHERITED (via int → stg → source)
├── email: → INHERITED (via int → stg → source)
├── full_name: → INHERITED (via int → stg)
├── enrichment_score: → INHERITED (via int)
└── contact_key: "Surrogate key" [NEW - added in mart]
```

---

## Plugin System

dbt-osmosis uses pluggy for extensible column name matching.

### Built-in Plugins

**FuzzyCaseMatching**
```python
# Generates case variants for column matching
# "user_id" → ["user_id", "USER_ID", "userId", "UserId"]
```

**FuzzyPrefixMatching**
```python
# Strips configured prefixes for inheritance matching
# With prefix="stg_": "stg_user_id" → "user_id"
```

### Custom Plugin Example

```python
# my_plugin.py
from dbt_osmosis.core.osmosis import hookimpl

class SuffixStripper:
    @hookimpl
    def get_candidates(self, name: str, node, context) -> list[str]:
        """Strip common suffixes for matching."""
        variants = [name]
        for suffix in ["_id", "_key", "_code"]:
            if name.endswith(suffix):
                variants.append(name[:-len(suffix)])
        return variants

# Register via setup.py entry_points:
# entry_points={
#     "dbt-osmosis": ["suffix_stripper = my_plugin:SuffixStripper"]
# }
```

---

## Maintenance Tasks

### Priority 1: Critical (Immediate)

#### 1.1 Dependency Updates
**Task:** Update and pin dependencies for security and compatibility
```
- dbt-core: Currently >=1.8,<1.11 → Test with dbt 1.10.x
- ruamel.yaml: >=0.17,<0.19 → Check for breaking changes
- pluggy: >=1.5.0,<2 → Generally stable
- mysql-mimic: >=2.5.7 → Check for security patches
- openai: Implicit dependency → Pin version explicitly
```

**Estimated Effort:** 4-8 hours  
**Risk Level:** Medium

#### 1.2 Python 3.13 Compatibility
**Task:** Test and fix compatibility with Python 3.13
- Update pyproject.toml: `requires-python = ">=3.10,<3.14"`
- Run full test suite on Python 3.13
- Fix any typing or async changes

**Estimated Effort:** 2-4 hours  
**Risk Level:** Low

### Priority 2: High (Within 1 Month)

#### 2.1 Test Coverage Expansion
**Task:** Increase test coverage for critical paths
- Current coverage: Unknown (estimate ~60%)
- Target: 80%+ for core/osmosis.py
- Focus areas:
  - Column knowledge graph building
  - YAML restructure operations
  - Adapter-specific column normalization

**Estimated Effort:** 16-24 hours  
**Risk Level:** Low

#### 2.2 Error Handling Improvements
**Task:** Improve error messages and recovery
- Add more specific exception types
- Improve error messages with actionable guidance
- Add recovery suggestions in CLI output

**Estimated Effort:** 8-12 hours  
**Risk Level:** Low

### Priority 3: Medium (Within 3 Months)

#### 3.1 Workbench Modernization
**Task:** Update Streamlit dependencies and UI
- Update streamlit: Currently >=1.20.0,<1.34.0 → Latest 1.x
- Update streamlit-ace and other components
- Improve error handling in workbench
- Add session state persistence

**Estimated Effort:** 16-24 hours  
**Risk Level:** Medium

#### 3.2 Performance Optimization
**Task:** Optimize for large projects
- Implement incremental manifest loading
- Add caching for column introspection
- Parallelize YAML file operations
- Profile and optimize hot paths

**Estimated Effort:** 24-40 hours  
**Risk Level:** Medium

#### 3.3 Documentation Improvements
**Task:** Update and expand documentation
- Create migration guides for major versions
- Add more configuration examples
- Document all CLI options comprehensively
- Create troubleshooting guide

**Estimated Effort:** 8-16 hours  
**Risk Level:** Low

### Priority 4: Low (Within 6 Months)

#### 4.1 Alternative LLM Support
**Task:** Add support for non-OpenAI LLMs
- Abstract LLM interface
- Add Anthropic Claude support
- Add local LLM support (Ollama)
- Make LLM provider configurable

**Estimated Effort:** 16-24 hours  
**Risk Level:** Low

#### 4.2 VS Code Extension Integration
**Task:** Improve integration with dbt-power-user
- Review current integration points
- Add any missing RPC endpoints
- Document integration patterns

**Estimated Effort:** 8-16 hours  
**Risk Level:** Low

### Maintenance Schedule

| Task Category | Frequency | Description |
|---------------|-----------|-------------|
| Dependency Audit | Monthly | Review CVEs, update minor versions |
| Test Execution | Per PR | Full test suite on all supported Python versions |
| dbt Compatibility | Per dbt Release | Test against new dbt-core releases |
| Performance Review | Quarterly | Profile and optimize if needed |
| Documentation Review | Quarterly | Update for accuracy |

### Breaking Change Checklist

Before any major version bump:
1. [ ] Update CHANGELOG.md with migration notes
2. [ ] Update version in pyproject.toml
3. [ ] Run full test suite
4. [ ] Test on sample dbt projects (jaffle_shop, etc.)
5. [ ] Update documentation site
6. [ ] Create GitHub release with notes
7. [ ] Publish to PyPI

---

## Troubleshooting Guide

### Common Issues

#### "Config key `dbt-osmosis: <path>` not set for model"

**Cause:** Model doesn't have +dbt-osmosis configured  
**Solution:** Add configuration in dbt_project.yml:
```yaml
models:
  your_project:
    +dbt-osmosis: "_{model}.yml"
```

#### Column introspection fails

**Cause:** Database connection issues or permissions  
**Solutions:**
1. Use `--catalog-path target/catalog.json` with pre-generated catalog
2. Run `dbt docs generate` first to create catalog
3. Check database permissions for metadata queries
4. Use `--disable-introspection` if no column sync needed

#### YAML formatting changes unexpectedly

**Cause:** Default ruamel.yaml settings  
**Solution:** Configure yaml_settings in dbt_project.yml:
```yaml
vars:
  dbt-osmosis:
    yaml_settings:
      width: 120
      indent: 2
```

#### Documentation not inheriting

**Cause:** Column name mismatch or empty descriptions  
**Solutions:**
1. Check column names match (case-sensitive for some adapters)
2. Verify source columns have non-empty descriptions
3. Use `--force-inherit-descriptions` to override existing
4. Check for fuzzy matching with `--add-progenitor-to-meta`

#### Pre-commit hook is slow

**Cause:** Full project parse on each commit  
**Solutions:**
1. Use path-based filtering: `files: ^models/`
2. Use FQN filtering: `args: [--fqn, staging]`
3. Use `--catalog-path` for faster column lookups
4. Consider running only on CI instead

---

## Appendix A: Key Data Structures

### ColumnInfo (from dbt-core)

```python
@dataclass
class ColumnInfo:
    name: str
    description: str | None
    data_type: str | None
    meta: dict[str, Any]
    tags: list[str]
    constraints: list[ColumnConstraint]
```

### SchemaFileLocation

```python
@dataclass
class SchemaFileLocation:
    target: Path              # Where file should be
    current: Path | None      # Where file currently is
    node_type: NodeType       # Model, Seed, or Source
    
    @property
    def is_valid(self) -> bool:
        return self.target == self.current
```

### RestructureOperation

```python
@dataclass
class RestructureOperation:
    file_path: Path                           # Target file
    content: dict[str, Any]                   # YAML content to write
    superseded_paths: dict[Path, list[Node]]  # Files to clean up
```

---

## Appendix B: Environment Variables

| Variable | Description |
|----------|-------------|
| `DBT_PROJECT_DIR` | Override project directory discovery |
| `DBT_PROFILES_DIR` | Override profiles directory discovery |
| `DBT_TARGET` | Default target profile |
| `DBT_PROFILE` | Default profile name |
| `DBT_THREADS` | Default thread count |
| `OPENAI_API_KEY` | Required for `--synthesize` feature |
| `OSMOSIS_LLM_MAX_SQL_CHARS` | Max SQL length for LLM prompts |

---

*Document generated for dbt-osmosis v1.1.17*  
*Last updated: December 2024*
