# dbt-osmosis - Technical Documentation

## Technical Architecture

This document provides a deep dive into the technical implementation of dbt-osmosis, covering its architecture, key algorithms, and design patterns.

## Core Architecture

### 1. Transformation Pipeline

The heart of dbt-osmosis is its transformation pipeline system, which allows operations to be composed and executed in sequence.

#### Pipeline Components

```python
@dataclass
class TransformOperation:
    """An operation to be run on a dbt manifest node."""
    func: t.Callable[..., t.Any]
    name: str
    _result: t.Any | None = None
    _context: t.Any | None = None
    _node: ResultNode | None = None
    _metadata: dict[str, t.Any] = field(init=False, default_factory=dict)
```

Key features:
- **Function composition**: Operations can be chained using `>>` operator
- **Metadata tracking**: Each operation tracks execution time, success/failure
- **Parallel execution**: Operations can run in parallel using multiprocessing
- **Atomic/commit modes**: Changes can be committed immediately, in batch, or deferred

#### Pipeline Execution Flow

```python
transform = (
    inject_missing_columns
    >> remove_columns_not_in_database
    >> inherit_upstream_column_knowledge
    >> sort_columns_as_configured
    >> synchronize_data_types
)

# Execute on all nodes
_ = transform(context=context)
```

### 2. Configuration System

#### Multi-Level Configuration

dbt-osmosis uses a hierarchical configuration system with four levels:

1. **Global CLI flags** (lowest priority)
2. **Folder-level** in `dbt_project.yml`
3. **Node-level** in SQL files via `config()`
4. **Column-level** in YAML metadata (highest priority)

#### Configuration Merging

```python
from dbt_osmosis.core.introspection import _get_setting_for_node

# Get setting with fallback chain
default_value = context.settings.skip_add_columns
node_specific = _get_setting_for_node(
    "skip-add-columns", 
    node, 
    name, 
    fallback=default_value
)
```

### 3. Documentation Inheritance

#### Knowledge Graph Construction

```python
def _build_column_knowledge_graph(context: YamlRefactorContext, node: ResultNode) -> dict[str, dict[str, t.Any]]:
    """Build a knowledge graph of column lineage and documentation."""
    # 1. Build ancestor tree
    ancestor_tree = _build_node_ancestor_tree(context, node)
    
    # 2. Traverse tree to collect knowledge
    column_knowledge_graph = {}
    for ancestor in ancestor_tree:
        ancestor_yaml = _get_node_yaml(context, ancestor)
        if not ancestor_yaml:
            continue
        
        # 3. Merge knowledge from ancestor
        for col_name, col_meta in ancestor_yaml.columns.items():
            if col_name not in column_knowledge_graph:
                column_knowledge_graph[col_name] = {}
            
            # 4. Inherit documentation
            if col_meta.description:
                column_knowledge_graph[col_name]["description"] = col_meta.description
            if col_meta.tags:
                column_knowledge_graph[col_name]["tags"] = col_meta.tags
            if col_meta.meta:
                column_knowledge_graph[col_name]["meta"] = col_meta.meta
    
    return column_knowledge_graph
```

#### Inheritance Algorithm

The inheritance system uses topological sorting to ensure:
1. Upstream models are processed before downstream models
2. Documentation flows from sources to final models
3. Circular dependencies are detected and handled

### 4. Restructuring Operations

#### Restructure Plan

```python
RestructureDeltaPlan = TypedDict('RestructureDeltaPlan', {
    'create': list[RestructureOperation],
    'move': list[RestructureOperation],
    'merge': list[RestructureOperation],
    'delete': list[RestructureOperation],
})
```

#### Operation Types

1. **Create**: Generate new YAML files for undocumented models
2. **Move**: Relocate YAML files to match configuration
3. **Merge**: Combine multiple YAML files into one
4. **Delete**: Remove YAML files that are no longer needed

#### Plan Execution

```python
def apply_restructure_plan(
    context: YamlRefactorContext,
    plan: RestructureDeltaPlan,
    confirm: bool = True,
) -> None:
    """Apply a restructure plan to the project."""
    
    # 1. Display plan
    pretty_print_plan(plan)
    
    # 2. Get user confirmation
    if confirm and not click.confirm("Apply these changes?", default=True):
        logger.info(":no_entry: User declined to apply changes.")
        return
    
    # 3. Execute operations
    for op_type, operations in plan.items():
        for op in operations:
            execute_operation(context, op)
```

## Key Algorithms

### 1. Column Matching

#### Normalization

```python
def normalize_column_name(column_name: str, adapter_type: str) -> str:
    """Normalize column names for consistent matching."""
    if adapter_type in ("snowflake", "bigquery", "redshift"):
        return column_name.lower()
    return column_name
```

#### Plugin System

```python
class PluginManager:
    def __init__(self):
        self._plugins: list[type[ColumnMatchingPlugin]] = []
    
    def register_plugin(self, plugin_class: type[ColumnMatchingPlugin]) -> None:
        """Register a new plugin."""
        self._plugins.append(plugin_class)
    
    def match_columns(
        self, 
        column_a: str, 
        column_b: str,
        config: dict[str, t.Any] | None = None
    ) -> bool:
        """Match columns using registered plugins."""
        for plugin_class in self._plugins:
            plugin = plugin_class(config)
            if plugin.match(column_a, column_b):
                return True
        return column_a == column_b
```

#### Built-in Plugins

1. **FuzzyCaseMatching**: Case-insensitive matching
2. **FuzzyPrefixMatching**: Strip prefixes before matching

### 2. Topological Sorting

```python
def _topological_sort(nodes: list[ResultNode]) -> list[ResultNode]:
    """Topologically sort nodes based on dependencies."""
    from dbt.graph import Graph
    
    # Build dependency graph
    graph = Graph()
    for node in nodes:
        graph.add_node(node.unique_id)
        for dep in node.depends_on_nodes:
            graph.add_edge(dep, node.unique_id)
    
    # Perform topological sort
    return [node for node in graph.topological_sort() if node in nodes]
```

### 3. YAML Synchronization

#### Delta Detection

```python
def sync_node_to_yaml(
    context: YamlRefactorContext,
    node: ResultNode,
    commit: bool = False,
) -> None:
    """Synchronize a node's columns with its YAML file."""
    
    # 1. Get current YAML
    current_yaml = get_current_yaml_path(context, node)
    
    # 2. Get target YAML path
    target_yaml = get_target_yaml_path(context, node)
    
    # 3. Detect changes
    changes = detect_yaml_changes(current_yaml, node.columns)
    
    # 4. Apply changes if needed
    if changes and commit:
        apply_yaml_changes(target_yaml, changes)
```

#### Change Detection Algorithm

```python
def detect_yaml_changes(
    current_yaml: dict,
    new_columns: dict[str, ColumnInfo]
) -> dict:
    """Detect differences between current and new column definitions."""
    changes = {"added": [], "removed": [], "modified": []}
    
    current_cols = {col["name"]: col for col in current_yaml.get("columns", [])}
    
    # Detect added columns
    for col_name, col_info in new_columns.items():
        if col_name not in current_cols:
            changes["added"].append(col_info)
    
    # Detect removed columns
    for col_name in current_cols:
        if col_name not in new_columns:
            changes["removed"].append(col_name)
    
    # Detect modified columns
    for col_name, col_info in new_columns.items():
        if col_name in current_cols:
            current = current_cols[col_name]
            if (current.get("description") != col_info.description or
                current.get("data_type") != col_info.data_type or
                current.get("tags") != col_info.tags or
                current.get("meta") != col_info.meta):
                changes["modified"].append(col_info)
    
    return changes
```

## Data Flow

### 1. Command Execution Flow

```
dbt-osmosis yaml refactor
    ↓
Parse CLI arguments
    ↓
Create DbtConfiguration
    ↓
Create YamlRefactorContext
    ↓
Create missing source YAMLs
    ↓
Draft restructure plan
    ↓
Apply restructure plan
    ↓
Execute transformation pipeline
    ↓
Commit changes to YAML files
```

### 2. Transformation Pipeline Flow

```
Start Pipeline
    ↓
inject_missing_columns
    ↓
remove_columns_not_in_database
    ↓
inherit_upstream_column_knowledge
    ↓
sort_columns_as_configured
    ↓
synchronize_data_types
    ↓
Commit YAML changes
```

### 3. Documentation Inheritance Flow

```
Process Node
    ↓
Build Ancestor Tree
    ↓
Traverse Ancestors
    ↓
Collect Column Knowledge
    ↓
Merge with Current Node
    ↓
Apply Inherited Documentation
```

## Performance Optimization

### 1. Parallel Processing

```python
from concurrent.futures import ProcessPoolExecutor

class YamlRefactorContext:
    def __init__(self, *args, **kwargs):
        self.pool = ProcessPoolExecutor()
        atexit.register(self.pool.shutdown)
    
    def process_nodes_in_parallel(self, nodes, func):
        """Process nodes in parallel using multiprocessing."""
        return list(self.pool.map(func, nodes))
```

### 2. Caching

```python
_COLUMN_LIST_CACHE = {}

def get_columns(context: YamlRefactorContext, node: ResultNode) -> dict[str, ColumnMetadata]:
    """Get columns for a node, with caching."""
    cache_key = (node.unique_id, context.project.runtime_cfg.credentials.type)
    
    if cache_key not in _COLUMN_LIST_CACHE:
        _COLUMN_LIST_CACHE[cache_key] = _fetch_columns_from_database(context, node)
    
    return _COLUMN_LIST_CACHE[cache_key]
```

### 3. Batch Operations

```python
def commit_yamls(
    context: YamlRefactorContext,
    batch_size: int = 100,
) -> None:
    """Commit YAML changes in batches."""
    nodes = list(context.project.manifest.nodes.values())
    
    for i in range(0, len(nodes), batch_size):
        batch = nodes[i:i + batch_size]
        
        # Process batch
        for node in batch:
            sync_node_to_yaml(context, node, commit=True)
        
        # Log progress
        logger.info(f"Committed batch {i//batch_size + 1}")
```

## Error Handling

### 1. Graceful Degradation

```python
@_transform_op("Synthesize Missing Documentation")
def synthesize_missing_documentation_with_openai(context, node):
    try:
        from dbt_osmosis.core.llm import generate_column_doc
        # ... synthesis logic
    except ImportError:
        logger.warning("OpenAI extra not installed, skipping documentation synthesis")
        return
    except Exception as e:
        logger.error(f"Failed to synthesize documentation: {e}")
        # Continue with other operations
```

### 2. Validation

```python
def validate_configuration(context: YamlRefactorContext) -> list[str]:
    """Validate configuration before execution."""
    errors = []
    
    # Check required configuration
    if not context.settings.project_dir:
        errors.append("project_dir is required")
    
    # Check database connection
    try:
        test_connection(context)
    except Exception as e:
        errors.append(f"Database connection failed: {e}")
    
    return errors
```

### 3. Transactional Operations

```python
def apply_restructure_plan(
    context: YamlRefactorContext,
    plan: RestructureDeltaPlan,
    confirm: bool = True,
) -> None:
    """Apply a restructure plan with rollback capability."""
    
    # Create backup
    backup_path = create_backup(context)
    
    try:
        # Apply changes
        execute_plan(context, plan)
        
        # Verify success
        verify_changes(context, plan)
        
    except Exception as e:
        # Rollback on failure
        logger.error(f"Plan execution failed: {e}")
        restore_from_backup(backup_path)
        raise
    
    finally:
        # Clean up backup
        cleanup_backup(backup_path)
```

## Integration Points

### 1. dbt Integration

#### Manifest Loading

```python
def create_dbt_project_context(config: DbtConfiguration) -> DbtProjectContext:
    """Create a dbt project context."""
    import dbt.config.project
    import dbt.tracking
    
    # Load dbt project
    project = dbt.config.project.Project(config.project_dir)
    
    # Load profiles
    profiles = load_profiles(config.profiles_dir)
    
    # Create runtime configuration
    runtime_cfg = project.credentials.from_profiles(profiles, config.target)
    
    # Load manifest
    manifest = load_manifest(project, runtime_cfg)
    
    return DbtProjectContext(
        project=project,
        profiles=profiles,
        runtime_cfg=runtime_cfg,
        manifest=manifest,
    )
```

#### SQL Compilation

```python
def compile_sql_code(project: DbtProjectContext, sql: str) -> ResultNode:
    """Compile dbt SQL code."""
    from dbt.task.sql import SqlCompileRunner
    
    # Create compilation runner
    runner = SqlCompileRunner(project.project, project.runtime_cfg)
    
    # Compile SQL
    node = runner.compile_node(sql)
    
    return node
```

### 2. Database Adapters

#### Column Introspection

```python
def get_columns(context: YamlRefactorContext, node: ResultNode) -> dict[str, ColumnMetadata]:
    """Get columns from database."""
    adapter = context.project.runtime_cfg.credentials.type
    
    if adapter == "snowflake":
        return _get_snowflake_columns(context, node)
    elif adapter == "bigquery":
        return _get_bigquery_columns(context, node)
    elif adapter == "postgres":
        return _get_postgres_columns(context, node)
    elif adapter == "duckdb":
        return _get_duckdb_columns(context, node)
    else:
        raise ValueError(f"Unsupported adapter: {adapter}")
```

#### Adapter-Specific Implementations

Each adapter has its own implementation for:
- Column type mapping
- Column name normalization
- Query execution
- Metadata extraction

### 3. LLM Integration

#### Documentation Synthesis

```python
def synthesize_missing_documentation_with_openai(
    context: YamlRefactorContext,
    node: ResultNode,
) -> None:
    """Synthesize missing documentation using OpenAI."""
    
    # Check if OpenAI is configured
    if not context.settings.openai_api_key:
        logger.warning("OpenAI API key not configured, skipping synthesis")
        return
    
    # Initialize OpenAI client
    client = OpenAI(api_key=context.settings.openai_api_key)
    
    # Generate documentation
    for column_name, column in node.columns.items():
        if not column.description:
            prompt = generate_column_prompt(column_name, node)
            response = client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": prompt}],
            )
            column.description = response.choices[0].message.content
```

## Testing Strategy

### 1. Unit Tests

```python
# tests/core/test_transforms.py

def test_inject_missing_columns():
    """Test column injection."""
    context = create_test_context()
    node = create_test_node()
    
    # Add some columns to database
    add_test_columns(context, node, ["col1", "col2"])
    
    # Run transformation
    inject_missing_columns(context, node)
    
    # Verify columns were added
    assert "col1" in node.columns
    assert "col2" in node.columns
```

### 2. Integration Tests

```python
# tests/integration/test_yaml_refactor.py

def test_yaml_refactor_integration():
    """Test full refactor workflow."""
    # Setup test project
    project_dir = create_test_project()
    
    # Run refactor
    context = create_context(project_dir)
    yaml_refactor(context)
    
    # Verify results
    assert_yaml_files_created()
    assert_columns_synchronized()
    assert_documentation_inherited()
```

### 3. End-to-End Tests

```python
# tests/e2e/test_jaffle_shop.py

def test_jaffle_shop_refactor():
    """Test refactor on jaffle_shop demo project."""
    # Clone jaffle_shop
    project_dir = clone_jaffle_shop()
    
    # Configure dbt-osmosis
    configure_project(project_dir)
    
    # Run refactor
    context = create_context(project_dir)
    yaml_refactor(context)
    
    # Verify all models have YAML files
    for model in get_all_models(project_dir):
        assert_yaml_exists(model)
```

## Deployment Considerations

### 1. Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/z3z1ma/dbt-osmosis
    rev: v1.1.5
    hooks:
      - id: dbt-osmosis
        files: ^models/
        args: [--target=prod, --dry-run, --check]
        additional_dependencies: [dbt-snowflake]
```

### 2. CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
jobs:
  dbt-osmosis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
      - name: Install dbt-osmosis
        run: pip install dbt-osmosis[workbench]
      - name: Run dbt-osmosis
        run: dbt-osmosis yaml refactor --check
      - name: Fail if changes detected
        if: failure()
        run: echo "dbt-osmosis detected changes that need to be committed"
```

### 3. Docker Container

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install dbt-osmosis
RUN pip install dbt-osmosis[workbench,openai]

# Copy project files
COPY . .

# Run dbt-osmosis
CMD ["dbt-osmosis", "yaml", "refactor"]
```

## Monitoring and Logging

### 1. Logging System

```python
import logging
from rich.logging import RichHandler

logger = logging.getLogger("dbt-osmosis")
logger.setLevel(logging.INFO)

# Rich formatting for console
console_handler = RichHandler(
    rich_tracebacks=True,
    tracebacks_show_locals=True,
)
console_handler.setFormatter(logging.Formatter("%message%s"))
logger.addHandler(console_handler)

# File logging
file_handler = logging.FileHandler("dbt-osmosis.log")
file_handler.setFormatter(logging.Formatter("%asctime)s - %name)s - %levelname)s - %message)s"))
logger.addHandler(file_handler)
```

### 2. Metrics Collection

```python
class Metrics:
    def __init__(self):
        self.timings = {}
        self.counters = {}
    
    def time(self, name: str, func: callable):
        """Time a function execution."""
        start = time.time()
        result = func()
        end = time.time()
        self.timings[name] = end - start
        return result
    
    def increment(self, name: str, amount: int = 1):
        """Increment a counter."""
        self.counters[name] = self.counters.get(name, 0) + amount
    
    def report(self):
        """Report metrics."""
        logger.info("Timings:")
        for name, duration in self.timings.items():
            logger.info(f"  {name}: {duration:.2f}s")
        
        logger.info("Counters:")
        for name, count in self.counters.items():
            logger.info(f"  {name}: {count}")
```

## Future Enhancements

### 1. Performance Improvements

- **Parallel YAML writing**: Write multiple YAML files simultaneously
- **Incremental processing**: Only process changed models
- **Smart caching**: Cache database schema information

### 2. Enhanced Features

- **Visualization**: Generate dependency graphs
- **Conflict resolution**: Interactive conflict resolution for merges
- **Advanced matching**: Machine learning-based column matching

### 3. Extensibility

- **Custom transformers**: Allow users to define custom transformations
- **Plugin marketplace**: Share and discover plugins
- **API endpoints**: REST API for integration with other tools

## Conclusion

This technical documentation provides a comprehensive overview of dbt-osmosis' architecture, algorithms, and implementation details. The system is designed with:

- **Modularity**: Components are loosely coupled and easily extensible
- **Performance**: Parallel processing and caching for efficiency
- **Reliability**: Comprehensive error handling and validation
- **Flexibility**: Multi-level configuration for different use cases

By understanding these technical details, developers can effectively maintain, extend, and customize dbt-osmosis to meet their specific requirements.
