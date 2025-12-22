# dbt-osmosis - Project Summary

## Executive Summary

**dbt-osmosis** is a powerful CLI tool that significantly enhances the developer experience for dbt (data build tool) projects. It automates schema YAML management, enables documentation inheritance across the data lineage, and provides an interactive workbench for dbt SQL development.

## What dbt-osmosis Does

### Core Value Proposition

dbt-osmosis solves three major pain points in dbt development:

1. **Schema YAML Management**: Automatically organizes, creates, and maintains YAML files according to declarative rules
2. **Documentation Inheritance**: Propagates column-level documentation from upstream models to downstream models
3. **Developer Productivity**: Provides a workbench for interactive SQL development and testing

### Key Features

| Feature | Description | Benefits |
|---------|-------------|----------|
| **YAML Refactor** | Complete workflow combining organization and documentation | Maintains consistent YAML structure across the project |
| **YAML Organize** | Restructures YAML files based on configuration | Ensures YAML files follow project conventions |
| **YAML Document** | Inherits documentation from upstream models | Reduces manual documentation effort |
| **SQL Run** | Executes dbt SQL statements | Quick testing and validation |
| **SQL Compile** | Compiles dbt SQL without execution | Validates Jinja logic |
| **Workbench** | Interactive Streamlit-based development environment | Faster iterative development |

## Architecture Overview

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    dbt-osmosis CLI                            │
└───────────────────┬───────────────────────┬───────────────────┘
                    │                       │
┌───────────────────▼───────┐ ┌─────────────────▼─────────────┐
│       YAML Management      │ │        SQL Operations          │
│  ┌─────────────────────┐   │ │  ┌─────────────────────┐       │
│  │  Restructuring      │   │ │  │  Compilation        │       │
│  │  Operations         │   │ │  │  Execution           │       │
│  └─────────────────────┘   │ │  └─────────────────────┘       │
│  ┌─────────────────────┐   │ │  ┌─────────────────────┐       │
│  │  Transformation     │   │ │  │  Workbench           │       │
│  │  Pipeline           │   │ │  │  (Streamlit)         │       │
│  └─────────────────────┘   │ │  └─────────────────────┘       │
└───────────────────┬───────┘ └─────────────────┬─────────────┘
                    │                           │
┌───────────────────▼───────────────────────▼───────────────────┐
│                    Core Services                            │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────┐  │
│  │  Configuration      │  │  Documentation      │  │  Plugins│  │
│  │  Management         │  │  Inheritance        │  │         │  │
│  └─────────────────────┘  └─────────────────────┘  └─────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### Key Components

1. **CLI Layer**: Click-based command-line interface
2. **Core Engine**: Transformation pipeline and configuration management
3. **YAML Management**: Restructuring, synchronization, and inheritance
4. **SQL Operations**: Compilation and execution
5. **Workbench**: Streamlit-based interactive environment
6. **Plugin System**: Extensible column matching and transformations

## How It Works

### 1. Configuration-Driven Approach

dbt-osmosis uses a declarative configuration approach in `dbt_project.yml`:

```yaml
models:
  my_project:
    +dbt-osmosis: "_{model}.yml"  # Default pattern
    
    staging:
      +dbt-osmosis: "{parent}.yml"  # Folder-based organization
      +dbt-osmosis-options:
        skip-add-columns: true
        sort-by: "alphabetical"
```

### 2. Transformation Pipeline

The core workflow follows this pipeline:

```
1. Create Missing Source YAMLs
    ↓
2. Draft Restructure Plan
    ↓
3. Apply Restructure Plan (Create/Move/Merge/Delete)
    ↓
4. Execute Transformations:
    • Inject Missing Columns
    • Remove Extra Columns
    • Inherit Upstream Documentation
    • Sort Columns
    • Synchronize Data Types
    ↓
5. Commit Changes to YAML Files
```

### 3. Documentation Inheritance

dbt-osmosis builds a knowledge graph to track column lineage:

```
Source Model (raw data)
    |
    v
Staging Model (cleaned)
    |
    v
Intermediate Model (transformed)
    |
    v
Final Model (business metrics)
```

When a column is documented in an upstream model, that documentation automatically flows to all downstream models that use that column.

## Usage Examples

### Basic Usage

```bash
# Run refactor on entire project
dbt-osmosis yaml refactor

# Dry run to preview changes
dbt-osmosis yaml refactor --dry-run

# Check if changes would be made
dbt-osmosis yaml refactor --check
```

### Advanced Usage

```bash
# Organize files only
dbt-osmosis yaml organize --auto-apply

# Document only
dbt-osmosis yaml document --force-inherit-descriptions

# Use catalog.json instead of live queries
dbt-osmosis yaml refactor --catalog-path target/catalog.json

# Filter by FQN
dbt-osmosis yaml refactor --fqn staging.sales
```

### SQL Operations

```bash
# Run a SQL query
dbt-osmosis sql run "SELECT * FROM {{ ref('customers') }} LIMIT 10"

# Compile SQL
dbt-osmosis sql compile "SELECT * FROM {{ ref('orders') }}"
```

### Workbench

```bash
# Start the workbench
dbt-osmosis workbench

# Custom host and port
dbt-osmosis workbench --host 0.0.0.0 --port 8080
```

## Configuration Options

### Multi-Level Configuration

dbt-osmosis supports configuration at multiple levels:

1. **Global CLI flags** (lowest priority)
2. **Folder-level** in `dbt_project.yml`
3. **Node-level** in SQL files via `config()`
4. **Column-level** in YAML metadata (highest priority)

### Common Options

| Option | Description | Default |
|--------|-------------|---------|
| `skip-add-columns` | Skip adding missing columns from database | false |
| `skip-add-data-types` | Skip adding data types | false |
| `skip-merge-meta` | Skip merging meta fields from upstream | false |
| `skip-add-tags` | Skip inheriting tags from upstream | false |
| `numeric-precision-and-scale` | Include precision/scale for numeric types | false |
| `string-length` | Include length for string types | false |
| `force-inherit-descriptions` | Override existing descriptions if empty | false |
| `output-to-lower` | Convert column names and types to lowercase | false |
| `sort-by` | Column sorting method (`database` or `alphabetical`) | `database` |

## Integration Points

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

### CI/CD Pipeline

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

### Docker

```dockerfile
FROM python:3.10-slim

WORKDIR /app

RUN pip install dbt-osmosis[workbench,openai]

COPY . .

CMD ["dbt-osmosis", "yaml", "refactor"]
```

## Benefits

### For Data Teams

1. **Consistency**: Ensures consistent YAML file structure across the project
2. **Documentation**: Automatically propagates documentation through the data lineage
3. **Productivity**: Reduces manual, repetitive tasks
4. **Quality**: Maintains data quality through automated validation

### For Individual Developers

1. **Faster Development**: Quick feedback through workbench
2. **Better Understanding**: Clear documentation inheritance shows data lineage
3. **Flexibility**: Multiple configuration levels allow customization
4. **Safety**: Dry-run and check modes prevent accidental changes

## Technical Highlights

### Key Technologies

- **Python 3.10+**: Modern Python features and type hints
- **Click**: CLI framework for command-line interface
- **dbt-core**: Integration with dbt's core functionality
- **Streamlit**: Web-based workbench for interactive development
- **Rich**: Beautiful console output and logging
- **Pydantic**: Data validation and settings management

### Performance Features

- **Parallel Processing**: Multiprocessing for large projects
- **Caching**: Column metadata caching to avoid repeated queries
- **Batch Operations**: Efficient YAML file commits
- **Incremental Processing**: Only processes changed models

### Extensibility

- **Plugin System**: Custom column matching algorithms
- **Transformation Pipeline**: Composable operations
- **Configuration Hierarchy**: Multi-level settings
- **Hook System**: Pre/post-processing hooks

## Project Status

### Current Version
- Latest stable release: v1.1.5
- Python versions: 3.10, 3.11, 3.12
- dbt versions: 1.8.0+

### Supported Adapters
- Snowflake
- BigQuery
- Redshift
- Postgres
- DuckDB
- SQLite
- And more (via dbt adapters)

### Community
- GitHub: https://github.com/z3z1ma/dbt-osmosis
- Documentation: https://z3z1ma.github.io/dbt-osmosis/
- PyPI: https://pypi.org/project/dbt-osmosis/

## Getting Started

### Installation

```bash
# Basic installation
pip install dbt-osmosis

# With workbench
pip install "dbt-osmosis[workbench]"

# With OpenAI support
pip install "dbt-osmosis[openai]"

# With all extras
pip install "dbt-osmosis[dev,workbench,openai]"
```

### Quick Start

1. **Configure** your `dbt_project.yml`:
   ```yaml
   models:
     your_project:
       +dbt-osmosis: "_{model}.yml"
   ```

2. **Run** dbt-osmosis:
   ```bash
   dbt-osmosis yaml refactor
   ```

3. **Review** the changes and commit them

## Best Practices

### 1. Start Small
- Begin with a subset of models using `--fqn`
- Test with `--dry-run` before applying changes
- Gradually expand to the entire project

### 2. Commit Configuration
- Store `dbt_project.yml` configuration in version control
- Document your YAML organization strategy
- Ensure team agreement on file naming conventions

### 3. Use Dry Runs
- Always use `--dry-run` first to preview changes
- Use `--check` in CI/CD to fail on unexpected changes
- Review changes before committing

### 4. Leverage Inheritance
- Document columns at the source/staging layer
- Let inheritance propagate documentation downstream
- Avoid duplicating documentation

### 5. Automate
- Add to pre-commit hooks for automatic YAML management
- Run in CI/CD to catch configuration drift
- Schedule regular refactors to keep YAMLs in sync

## Future Directions

### Potential Enhancements

1. **Enhanced LLM Integration**: Better documentation synthesis
2. **Visualization**: Dependency graphs and lineage visualization
3. **Conflict Resolution**: Interactive merge conflict resolution
4. **Performance**: Faster processing for very large projects
5. **More Adapters**: Support for additional dbt adapters
6. **Custom Transformers**: User-defined transformation operations

### Roadmap

- Improved error handling and recovery
- Better documentation and examples
- Enhanced workbench features
- Performance optimizations for large projects
- Additional integration options

## Conclusion

dbt-osmosis is a powerful tool that significantly enhances the dbt development experience by:

- **Automating** repetitive YAML management tasks
- **Ensuring** consistent documentation across the data lineage
- **Providing** an interactive workbench for faster development
- **Supporting** customization through a flexible configuration system
- **Integrating** seamlessly with dbt workflows and CI/CD pipelines

By using dbt-osmosis, data teams can maintain better documentation, reduce manual errors, and focus on building high-quality data models.

## Additional Resources

- [Comprehensive Documentation](DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md)
- [Technical Documentation](DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md)
- [Official Documentation Site](https://z3z1ma.github.io/dbt-osmosis/)
- [GitHub Repository](https://github.com/z3z1ma/dbt-osmosis)
- [PyPI Package](https://pypi.org/project/dbt-osmosis/)
