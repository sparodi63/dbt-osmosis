# dbt-osmosis Workbench - Complete Guide

## Overview

**dbt-osmosis Workbench** is an interactive development environment for dbt (data build tool) that provides a Streamlit-based interface for editing, compiling, and testing dbt SQL models in real-time. It's designed to enhance the dbt development experience by offering features similar to the osmosis server + power-user combo without requiring VS Code.

## Key Features

### 1. Interactive SQL Editor
- **Real-time compilation**: See compiled SQL as you type
- **Multiple layouts**: Side-by-side or pivoted full-screen editor
- **Model selection**: Quickly load any dbt model from your project
- **Save functionality**: Persist changes back to your model files

### 2. Query Tester
- **Execute queries**: Run your SQL against the selected target database
- **Instant feedback**: See results immediately
- **Multiple execution options**:
  - Ctrl+Enter or Cmd+Shift+S to run queries
  - Ctrl+S to save models

### 3. Data Profiler
- **Automated profiling**: Generate comprehensive data profiles using pandas-profiling
- **Statistical analysis**: Get insights into your data distribution, missing values, correlations, etc.
- **Visualizations**: Interactive charts and graphs

### 4. Profile Management
- **Target selection**: Switch between different dbt profiles/targets
- **Context-aware compilation**: Compile and execute against the selected target

### 5. Additional Features
- **RSS Feed**: Hacker News feed integrated in the interface
- **Useful Links**: Quick access to documentation and resources
- **Theme support**: Light and dark mode options

## Installation

To use the workbench, you need to install dbt-osmosis with the workbench extra:

```bash
pip install "dbt-osmosis[workbench]"
```

The workbench requires additional dependencies:
- `streamlit` - The web framework
- `streamlit-elements` - For interactive UI components
- `pandas-profiling` - For data profiling
- `feedparser` - For RSS feed functionality

## Usage

### Basic Command

```bash
dbt-osmosis workbench [--project-dir] [--profiles-dir] [--host] [--port]
```

### Common Options

- `--project-dir`: Path to your dbt project directory
- `--profiles-dir`: Path to your dbt profiles directory
- `--host`: Host to run the Streamlit app on (default: localhost)
- `--port`: Port to run the Streamlit app on (default: 8501)

### Example

```bash
# Start workbench with default settings
dbt-osmosis workbench

# Start with specific project and profiles
cd /path/to/your/dbt/project
dbt-osmosis workbench --profiles-dir ~/.dbt/profiles.yml

# Start on a different port
dbt-osmosis workbench --port 8502
```

## Interface Overview

### Sidebar

The left sidebar contains several expandable sections:

1. **💡 Models**
   - Select a model to edit (or use "SCRATCH" for temporary queries)
   - Save button (appears when editing a real model)
   - Revert button (resets to original content)

2. **💁 Profiles**
   - Select which dbt target/profile to use
   - Shows current target

3. **📝 Query Template**
   - Customize the SQL template used for execution
   - `{sql}` placeholder is replaced with compiled SQL

4. **Notes**
   - Refresh instructions
   - Other helpful tips

### Main Dashboard

The main area is divided into resizable panels:

1. **Editor Panel** (top-left)
   - SQL/Jinja editor with syntax highlighting
   - Tabs for different file types
   - Compilation status

2. **Renderer Panel** (top-right)
   - Shows compiled SQL output
   - Side-by-side comparison with your source

3. **Preview Panel** (middle-left)
   - Query execution results
   - Tabular display of data
   - Execution status and errors

4. **Profiler Panel** (bottom-left)
   - Data profiling reports
   - Statistical analysis
   - Visualizations

5. **Feed Panel** (bottom-right)
   - Hacker News RSS feed
   - Quick access to industry news

## Workflow

### 1. Starting a New Query
1. Select "SCRATCH" from the model dropdown
2. Type or paste your SQL/Jinja code in the editor
3. Press Ctrl+Enter to compile and see the result
4. Press Ctrl+Shift+Enter to execute the query

### 2. Editing an Existing Model
1. Select a model from the dropdown
2. The model content is automatically loaded
3. Make your changes in the editor
4. Press Ctrl+Enter to compile
5. Press Ctrl+Shift+Enter to test
6. Click "💾 - Save" to persist changes to disk

### 3. Using Different Targets
1. Expand the "💁 Profiles" section
2. Select a different target from the radio buttons
3. The compilation and execution will use the new target

### 4. Profiling Data
1. Execute a query to get results
2. Go to the "Profiler" panel
3. Click "Run Profile" to generate a report
4. View the comprehensive data profile

## Keyboard Shortcuts

- **Ctrl+Enter**: Compile the current SQL
- **Ctrl+Shift+Enter**: Execute the current query
- **Ctrl+S**: Save the current model (if not in SCRATCH mode)
- **Command+S** (Mac): Same as Ctrl+S
- **Command+Shift+S** (Mac): Same as Ctrl+Shift+Enter

## Tips and Best Practices

1. **Refresh the page** if you've made changes to your dbt project files (models, macros, etc.) that aren't reflected in the workbench

2. **Use SCRATCH mode** for testing snippets without affecting your actual models

3. **Save frequently** when editing real models to avoid losing work

4. **Test with different targets** to ensure your models work across environments

5. **Use the profiler** to understand your data better before building complex transformations

6. **Leverage the query template** to wrap your queries with additional SQL if needed

## Technical Details

### Architecture

The workbench is built using:
- **Streamlit**: Web framework for the UI
- **Streamlit Elements**: Interactive dashboard components
- **Pandas Profiling**: Data profiling library
- **dbt-osmosis Core**: For dbt project context, compilation, and execution

### Key Components

1. **app.py**: Main application file that orchestrates the workbench
2. **components/**: Contains individual UI components
   - `editor.py`: SQL editor with compilation
   - `renderer.py`: Compiled SQL display
   - `preview.py`: Query execution and results
   - `profiler.py`: Data profiling
   - `dashboard.py`: Layout management
   - `feed.py`: RSS feed

### State Management

The workbench uses Streamlit's session state to maintain:
- Current query
- Compiled SQL
- Query results
- Selected model
- Selected target
- Profile reports

## Troubleshooting

### Common Issues

1. **Workbench won't start**
   - Ensure all dependencies are installed: `pip install "dbt-osmosis[workbench]"`
   - Check that your dbt project is valid
   - Verify your profiles.yml is correctly configured

2. **Compilation errors**
   - Make sure all your dbt dependencies are installed
   - Check that your project can be parsed by dbt
   - Verify that referenced models/macros exist

3. **Execution errors**
   - Ensure your database connection is working
   - Check that the target profile has valid credentials
   - Verify the database is accessible from your environment

4. **Performance issues**
   - Large queries may take time to execute
   - Profiling large datasets can be memory-intensive
   - Consider using `LIMIT` clauses for testing

### Debugging

- Check the terminal output for error messages
- Look at the Streamlit logs
- Try running `dbt parse` to verify your project
- Test your database connection with `dbt debug`

## Advanced Usage

### Custom Query Templates

You can customize how queries are executed by modifying the query template:

```sql
-- Example template that adds query timing
EXPLAIN ANALYZE {sql}
```

Or for adding transaction control:

```sql
BEGIN TRANSACTION;
{sql}
ROLLBACK;
```

### Integration with CI/CD

While the workbench is primarily for development, you can use other dbt-osmosis commands in your CI/CD pipeline:

```yaml
# Example GitHub Actions workflow
- name: Run dbt-osmosis
  run: |
    dbt-osmosis yaml refactor --project-dir . --profiles-dir ~/.dbt
    dbt-osmosis yaml document --project-dir . --profiles-dir ~/.dbt
```

## Demo and Examples

A live demo of the workbench is available at:
https://dbt-osmosis-playground.streamlit.app/

This demo uses the jaffle_shop example project to showcase the workbench features.

## Conclusion

The dbt-osmosis workbench provides a powerful, interactive environment for dbt development that accelerates the feedback loop and improves productivity. By combining real-time compilation, execution, and data profiling in a single interface, it enables data engineers to work more efficiently and with greater confidence.

For more information, visit the [dbt-osmosis documentation site](https://z3z1ma.github.io/dbt-osmosis/).
