# dbt-osmosis - Comprehensive Documentation

## Overview

**dbt-osmosis** is a powerful CLI tool that enhances the developer experience for dbt (data build tool) projects. It provides automated schema YAML management, documentation inheritance, and a workbench for dbt Jinja SQL development.

## Core Features

### 1. Automated Schema YAML Management

dbt-osmosis provides three main commands for managing YAML files:

#### a. `yaml refactor` - Complete Workflow
Combines organization and documentation in the correct order:
- Creates or moves YAML files based on configuration
- Ensures columns match the database schema
- Inherits descriptions and metadata from upstream models
- Reorders columns if configured

#### b. `yaml organize` - File Organization
- Restructures YAML files based on declarative rules in `dbt_project.yml`
- Bootstraps missing YAML files for undocumented models
- Moves or merges existing YAML files according to configuration

#### c. `yaml document` - Documentation Inheritance
- Adds missing columns from the database
- Removes columns no longer in the database
- Inherits tags, descriptions, and meta fields from upstream models
- Reorders columns based on configuration

### 2. SQL Operations

#### `sql run` - Execute SQL
Runs dbt SQL statements (including Jinja) and writes results to stdout.

#### `sql compile` - Compile SQL
Compiles dbt SQL statements (including Jinja) without executing them.

### 3. Workbench - Interactive Development Environment
A Streamlit-powered workbench for:
- Side-by-side dbt model editing and compilation
- Real-time query execution and testing
- Data profiling
- Theme customization (light/dark mode)

## Architecture Overview

### Core Components

```
src/dbt_osmosis/
├── cli/                  # Command-line interface
│   └── main.py           # CLI entry point with Click-based commands
├── core/                 # Core functionality
│   ├── config.py         # Configuration management
│   ├── inheritance.py    # Documentation inheritance logic
│   ├── introspection.py  # Database introspection
│   ├── osmosis.py        # Main exports and backwards compatibility
│   ├── path_management.py # YAML file path management
│   ├── plugins.py        # Plugin system for custom matching
│   ├── restructuring.py  # File restructuring operations
│   ├── schema/           # Schema parsing and writing
│   ├── settings.py       # Settings and context management
│   ├── sql_operations.py # SQL compilation and execution
│   ├── sync_operations.py # Synchronization operations
│   └── transforms.py     # Transformation pipelines
├── sql/                  # SQL-related utilities
├── workbench/            # Streamlit workbench
│   ├── app.py           # Main workbench application
│   └── components/      # Workbench UI components
└── __init__.py          # Package initialization
```

### Key Concepts

#### 1. Configuration System

dbt-osmosis uses a multi-level configuration system:

**Level 1: Global CLI Flags**
```bash
dbt-osmosis yaml refactor \
  --skip-add-columns \
  --skip-add-data-types \
  --numeric-precision-and-scale
```

**Level 2: Folder-Level in dbt_project.yml**
```yaml
models:
  my_project:
    staging:
      +dbt-osmosis: "{parent}.yml"
      +dbt-osmosis-options:
        skip-add-columns: true
        sort-by: "alphabetical"
```

**Level 3: Node-Level in SQL Files**
```jinja
{{ config(
    materialized='incremental',
    dbt_osmosis_options={
      "skip-add-data-types": True,
      "sort-by": "alphabetical"
    }
) }}
```

**Level 4: Column-Level in YAML**
```yaml
columns:
  - name: tricky_column
    meta:
      dbt-osmosis-skip-add-data-types: true
```

#### 2. Transformation Pipeline

The core operations are composed using function composition:

```python
transform = (
    inject_missing_columns
    >> remove_columns_not_in_database
    >> inherit_upstream_column_knowledge
    >> sort_columns_as_configured
    >> synchronize_data_types
)
```

Key transformations:
- **inject_missing_columns**: Adds columns from database that are missing in YAML
- **remove_columns_not_in_database**: Removes columns that no longer exist in database
- **inherit_upstream_column_knowledge**: Propagates documentation from upstream models
- **sort_columns_as_configured**: Reorders columns based on configuration
- **synchronize_data_types**: Updates data types to match database

#### 3. Documentation Inheritance

dbt-osmosis builds a knowledge graph to track column lineage:

```
Upstream Model (source)
    |
    v
Intermediate Model
    |
    v
Downstream Model (target)
```

When a column is documented in an upstream model, that documentation is inherited by downstream models that use that column.

#### 4. Restructuring Plan

Before making changes, dbt-osmosis creates a plan:

```python
RestructureDeltaPlan = TypedDict('RestructureDeltaPlan', {
    'create': list[RestructureOperation],
    'move': list[RestructureOperation],
    'merge': list[RestructureOperation],
    'delete': list[RestructureOperation],
})
```

This plan shows:
- Files to create
- Files to move
- Files to merge
- Files to delete

Users can review and confirm before changes are applied.

## Configuration Reference

### Basic Configuration

```yaml
title: 'My dbt Project'
version: '1.0.0'
config-version: 2

models:
  my_project:
    # Default YAML file pattern for all models
    +dbt-osmosis: "_{model}.yml"
    
    # Folder-specific configurations
    staging:
      +dbt-osmosis: "{parent}.yml"
      +dbt-osmosis-options:
        skip-add-columns: true
        sort-by: "alphabetical"
    
    intermediate:
      +dbt-osmosis: "{node.config[materialized]}/{model}.yml"
      +dbt-osmosis-options:
        skip-add-tags: true
        output-to-lower: true

seeds:
  my_project:
    +dbt-osmosis: "_schema.yml"

vars:
  dbt-osmosis:
    sources:
      salesforce:
        path: "staging/salesforce/source.yml"
        schema: "salesforce_v2"
      marketo: "staging/customer/marketo.yml"
    
    column_ignore_patterns:
      - "_FIVETRAN_SYNCED"
      - ".*__key__.namespace"
```

### Configuration Options

| Option | Type | Description | Default |
|--------|------|-------------|---------|
| `skip-add-columns` | bool | Skip adding missing columns from database | false |
| `skip-add-source-columns` | bool | Skip adding columns to source YAMLs | false |
| `skip-add-data-types` | bool | Skip adding data types | false |
| `skip-merge-meta` | bool | Skip merging meta fields from upstream | false |
| `skip-add-tags` | bool | Skip inheriting tags from upstream | false |
| `numeric-precision-and-scale` | bool | Include precision and scale for numeric types | false |
| `string-length` | bool | Include length for string types | false |
| `force-inherit-descriptions` | bool | Override existing descriptions if empty | false |
| `output-to-lower` | bool | Convert column names and types to lowercase | false |
| `sort-by` | string | Column sorting method (`database` or `alphabetical`) | `database` |
| `add-progenitor-to-meta` | bool | Add progenitor information to meta | false |
| `add-inheritance-for-specified-keys` | list | Additional keys to inherit | [] |

## Usage Examples

### Basic Usage

```bash
# Run refactor on entire project
dbt-osmosis yaml refactor

# Run on specific models
dbt-osmosis yaml refactor model1 model2 model3

# Dry run (preview changes without applying)
dbt-osmosis yaml refactor --dry-run

# Check if changes would be made (exit with code 1 if changes detected)
dbt-osmosis yaml refactor --check
```

### Advanced Usage

```bash
# Organize files only (no documentation)
dbt-osmosis yaml organize --auto-apply

# Document only (no file organization)
dbt-osmosis yaml document --force-inherit-descriptions

# Use catalog.json instead of querying database
dbt-osmosis yaml refactor --catalog-path target/catalog.json

# Skip database connection entirely
dbt-osmosis yaml refactor --disable-introspection

# Filter by FQN
dbt-osmosis yaml refactor --fqn staging.sales
```

### SQL Operations

```bash
# Run a SQL query
dbt-osmosis sql run "SELECT * FROM {{ ref('customers') }} LIMIT 10"

# Compile SQL without executing
dbt-osmosis sql compile "SELECT * FROM {{ ref('orders') }}"
```

### Workbench

```bash
# Start the workbench
dbt-osmosis workbench

# Custom host and port
dbt-osmosis workbench --host 0.0.0.0 --port 8080
```

## Plugin System

dbt-osmosis includes a plugin system for custom column matching:

### Built-in Plugins

1. **FuzzyCaseMatching**: Matches columns ignoring case differences
2. **FuzzyPrefixMatching**: Matches columns with configurable prefix stripping

### Creating Custom Plugins

```python
from dbt_osmosis.core.plugins import get_plugin_manager

class CustomMatchingPlugin:
    def __init__(self, config=None):
        self.config = config or {}
    
    def match(self, column_a, column_b):
        # Implement custom matching logic
        return column_a.lower() == column_b.lower()

# Register the plugin
plugin_manager = get_plugin_manager()
plugin_manager.register_plugin(CustomMatchingPlugin)
```

## Integration with dbt

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/z3z1ma/dbt-osmosis
    rev: v1.1.5
    hooks:
      - id: dbt-osmosis
        files: ^models/
        args: [--target=prod]
        additional_dependencies: [dbt-snowflake]
```

### CI/CD Integration

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run dbt-osmosis
        run: dbt-osmosis yaml refactor --check
```

## Best Practices

### 1. Start with Organization

Begin with `yaml organize` to establish your YAML file structure before adding documentation.

### 2. Use Dry Runs

Always use `--dry-run` first to preview changes before applying them.

### 3. Commit Configuration

Store your `dbt_project.yml` configuration in version control so the team agrees on YAML file locations.

### 4. Gradual Adoption

Start with a subset of models using `--fqn` before applying to the entire project.

### 5. Review Generated Documentation

While dbt-osmosis can synthesize documentation with OpenAI, always review and refine auto-generated content.

## Troubleshooting

### Common Issues

**Issue: Connection errors**
- Ensure your database credentials are correct in `profiles.yml`
- Use `--catalog-path` to avoid live queries if offline

**Issue: No changes detected**
- Check your configuration in `dbt_project.yml`
- Verify the models exist in your dbt project
- Use `--dry-run` to see what would change

**Issue: Permission errors**
- Ensure you have write permissions to the models directory
- Check file permissions on existing YAML files

**Issue: Column matching failures**
- Use plugins for custom matching logic
- Check for case sensitivity issues
- Review column naming conventions

## Development

### Setting Up for Development

```bash
# Clone the repository
git clone https://github.com/z3z1ma/dbt-osmosis.git
cd dbt-osmosis

# Install in development mode
pip install -e ".[dev,workbench,openai]"

# Run tests
pytest tests/
```

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Submit a pull request

## Future Enhancements

Potential areas for improvement:
- Enhanced LLM integration for better documentation synthesis
- Support for more dbt adapters
- Improved performance for large projects
- Better conflict resolution for merge scenarios
- Visualization of documentation inheritance graph

## Conclusion

dbt-osmosis significantly enhances the dbt development experience by:
- Automating YAML file management
- Ensuring documentation consistency
- Providing an interactive workbench
- Supporting customization through plugins
- Integrating seamlessly with dbt workflows

By using dbt-osmosis, teams can maintain better documentation, reduce manual errors, and focus on building high-quality data models.
