# dbt-osmosis Workbench User Guide

## Table of Contents

1. [Introduction](#1-introduction)
2. [Installation](#2-installation)
3. [Getting Started](#3-getting-started)
4. [Interface Overview](#4-interface-overview)
5. [Working with SQL](#5-working-with-sql)
6. [Query Execution](#6-query-execution)
7. [Data Profiling](#7-data-profiling)
8. [Keyboard Shortcuts](#8-keyboard-shortcuts)
9. [Tips & Best Practices](#9-tips--best-practices)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Introduction

### What is the Workbench?

The **dbt-osmosis Workbench** is an interactive web-based development environment for writing and testing dbt SQL queries. It provides a real-time compilation view, query execution capabilities, and data profiling tools - all in a single, unified interface.

### Key Features

| Feature | Description |
|---------|-------------|
| **SQL Editor** | Monaco-based editor with syntax highlighting and dbt Jinja support |
| **Real-time Compilation** | See your compiled SQL instantly as you type |
| **Query Execution** | Run queries against your data warehouse directly |
| **Data Profiling** | Statistical analysis of query results using pandas-profiling |
| **Model Browser** | Navigate and edit existing dbt models |
| **Profile Switching** | Switch between dbt targets on the fly |
| **Dark/Light Mode** | Theme switching for comfortable viewing |

### Use Cases

- **Rapid SQL Development**: Write and test dbt SQL without running `dbt compile`
- **Query Debugging**: See exactly what SQL dbt generates from your Jinja templates
- **Data Exploration**: Execute ad-hoc queries and profile results
- **Model Development**: Edit existing models with instant feedback

---

## 2. Installation

### Prerequisites

- Python 3.10, 3.11, or 3.12
- A configured dbt project with `profiles.yml`
- Database credentials set up in your profile

### Install with Workbench Dependencies

```bash
pip install "dbt-osmosis[workbench]"
```

This installs the additional packages required:
- `streamlit` - Web framework
- `streamlit-ace` - Code editor
- `streamlit-elements-fluence` - Dashboard components
- `ydata-profiling` - Data profiling
- `feedparser` - RSS feed parsing
- `dbt-duckdb` - DuckDB adapter (for demos)

### Verify Installation

```bash
dbt-osmosis workbench --help
```

---

## 3. Getting Started

### Starting the Workbench

Navigate to your dbt project directory and run:

```bash
cd /path/to/your/dbt/project
dbt-osmosis workbench
```

The workbench will start and open in your default browser at `http://localhost:8501`.

### Command Options

```bash
dbt-osmosis workbench [OPTIONS]

Options:
  --project-dir    Path to dbt project (default: current directory)
  --profiles-dir   Path to profiles.yml directory (default: ~/.dbt)
  --host           Server host (default: localhost)
  --port           Server port (default: 8501)
```

### Example: Custom Configuration

```bash
# Use a specific project and profile directory
dbt-osmosis workbench \
  --project-dir /path/to/project \
  --profiles-dir /path/to/profiles \
  --port 8080
```

### First Launch

When you first open the Workbench:

1. **dbt Context Loaded**: Your dbt project is parsed and loaded
2. **Default Query**: A sample query appears in the editor
3. **Models Available**: All your project's models are accessible via the sidebar
4. **Profile Active**: Your default target is automatically selected

---

## 4. Interface Overview

### Layout

The Workbench uses a draggable dashboard layout with five main panels:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           dbt-osmosis Workbench                              │
├─────────────────────────────────┬───────────────────────────────────────────┤
│                                 │                                            │
│       SQL EDITOR                │         COMPILED SQL                       │
│   (Write your dbt SQL here)     │    (See the rendered output)               │
│                                 │                                            │
│   [SQL] [YAML]                  │                                            │
│                                 │                                            │
├─────────────────────────────────┴───────────────────────────────────────────┤
│                                                                              │
│                        QUERY PREVIEW                                         │
│               (Query results displayed here)                                 │
│                                                                              │
├─────────────────────────────────────────────────────┬────────────────────────┤
│                                                     │                        │
│           PANDAS PROFILER                           │    HACKER NEWS FEED    │
│    (Statistical data profiling)                     │    (Tech news)         │
│                                                     │                        │
└─────────────────────────────────────────────────────┴────────────────────────┘
```

### Panels Description

| Panel | Location | Purpose |
|-------|----------|---------|
| **SQL Editor** | Top-left | Write dbt SQL with Jinja templates |
| **Compiled SQL** | Top-right | View the compiled (pure SQL) output |
| **Query Preview** | Middle | Display query execution results |
| **Pandas Profiler** | Bottom-left | Generate statistical profiles of data |
| **Hacker News Feed** | Bottom-right | Tech news feed (for inspiration!) |

### Sidebar

The left sidebar contains:

1. **Models Selector**: Choose a model to edit or use SCRATCH mode
2. **Profiles/Targets**: Switch between dbt targets
3. **Query Template**: Customize the execution wrapper

### Resizing Panels

All panels are **draggable and resizable**:
- **Drag**: Click and hold the title bar to move a panel
- **Resize**: Drag panel edges to resize

---

## 5. Working with SQL

### The SQL Editor

The SQL Editor supports full dbt Jinja syntax:

```sql
-- Use ref() to reference other models
select * from {{ ref('stg_customers') }}

-- Use source() to reference sources
select * from {{ source('raw', 'customers') }}

-- Use Jinja loops and variables
{% set columns = ['id', 'name', 'email'] %}
select
    {% for col in columns %}
    {{ col }}{% if not loop.last %},{% endif %}
    {% endfor %}
from {{ ref('my_model') }}

-- Use config blocks
{{ config(materialized='table') }}
select * from {{ ref('upstream') }}
```

### Editor Tabs

| Tab | Purpose |
|-----|---------|
| **SQL** | Write your dbt SQL code |
| **YAML** | Edit YAML (for schema documentation) |

### Loading Existing Models

1. Open the **Models** expander in the sidebar
2. Use the dropdown to select a model
3. The model's SQL is loaded into the editor
4. Make your changes
5. Click **Save** to persist changes to disk

### SCRATCH Mode

When "SCRATCH" is selected:
- You can write ad-hoc queries
- Changes are NOT saved to disk
- Perfect for experimentation

### Real-time Compilation

Toggle **Realtime Compilation** at the bottom of the editor:
- **ON**: SQL compiles automatically as you type
- **OFF**: SQL compiles when you press Ctrl+Enter

---

## 6. Query Execution

### Running Queries

1. Write your SQL in the editor
2. Verify the compiled output in the "Compiled SQL" panel
3. Click **Run Query** or press `Ctrl+Shift+Enter`
4. Results appear in the Query Preview panel

### Query Template

The Query Template (in sidebar) wraps your compiled SQL:

```sql
-- Default template
select * from ({sql}) as _query limit 200
```

You can customize this:

```sql
-- Custom template examples

-- Get first 1000 rows
select * from ({sql}) as _query limit 1000

-- Only get count
select count(*) as total from ({sql}) as _query

-- Sample data
select * from ({sql}) as _query tablesample(1000 rows)
```

### Query States

| State | Indicator | Meaning |
|-------|-----------|---------|
| **Running** | Spinner | Query is executing |
| **Success** | Data Grid | Results available |
| **Error** | Error message | Query failed |
| **Empty** | "No results" | No data returned |

### Results Grid

The results grid supports:
- **Sorting**: Click column headers
- **Pagination**: Navigate through large result sets
- **Resize columns**: Drag column borders

---

## 7. Data Profiling

### What is Data Profiling?

Data profiling provides statistical analysis of your query results:
- Column types and distributions
- Missing values
- Correlations
- Histograms
- Outliers

### Using the Profiler

1. **Execute a query** (results must be available)
2. Click **Profile Results** in the Profiler panel
3. Wait for the report to generate
4. Browse the interactive HTML report

### Profile Features

| Section | Content |
|---------|---------|
| **Overview** | Dataset stats, variable types |
| **Variables** | Per-column analysis |
| **Interactions** | Correlation matrix |
| **Correlations** | Numeric correlations |
| **Missing Values** | Missing data patterns |

### Performance Note

Profiling large datasets can be slow. Consider:
- Limiting your query results (adjust Query Template)
- Using `LIMIT` in your SQL

---

## 8. Keyboard Shortcuts

### Editor Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl + Enter` | Compile SQL |
| `Cmd + S` (Mac) | Compile SQL |
| `Ctrl + Shift + Enter` | Run Query |
| `Cmd + Shift + S` (Mac) | Run Query |

### Monaco Editor Shortcuts

The SQL editor uses Monaco (same as VS Code), so standard shortcuts work:

| Shortcut | Action |
|----------|--------|
| `Ctrl + Z` | Undo |
| `Ctrl + Shift + Z` | Redo |
| `Ctrl + F` | Find |
| `Ctrl + H` | Find and Replace |
| `Ctrl + /` | Toggle Comment |
| `Ctrl + D` | Select next occurrence |
| `Alt + Up/Down` | Move line up/down |

---

## 9. Tips & Best Practices

### Workflow Tips

1. **Start with SCRATCH**: Use SCRATCH mode for experimentation before modifying models

2. **Compile First**: Always verify the compiled SQL before running

3. **Use the Template**: Adjust the query template to limit results during development

4. **Save Frequently**: Use Save button to persist changes to your model files

5. **Profile Selectively**: Only profile when you need statistical insights

### Performance Tips

1. **Limit Results**: Keep query results under 10,000 rows for best performance

2. **Disable Realtime**: Turn off realtime compilation for complex queries

3. **Refresh for Updates**: Refresh the page after changing files externally

### dbt Integration Tips

1. **Target Switching**: Switch targets in the sidebar to test against different environments

2. **Revert Changes**: Use the Revert button to discard unsaved changes

3. **Macro Testing**: You can call custom macros in your queries

---

## 10. Troubleshooting

### Common Issues

#### Workbench Won't Start

**Symptom**: Error when running `dbt-osmosis workbench`

**Solutions**:
```bash
# Ensure workbench extras are installed
pip install "dbt-osmosis[workbench]"

# Check for port conflicts
lsof -i :8501

# Try a different port
dbt-osmosis workbench --port 8502
```

#### Models Not Appearing

**Symptom**: Model list is empty or missing models

**Solutions**:
1. Ensure you're in the correct project directory
2. Run `dbt parse` to generate manifest
3. Refresh the browser page

#### Compilation Errors

**Symptom**: Compiled SQL shows error message

**Common Causes**:
- Missing models in ref() calls
- Undefined variables
- Syntax errors in Jinja

**Solutions**:
1. Check the error message for details
2. Verify the referenced model exists
3. Run `dbt compile` in terminal to see full error

#### Query Execution Fails

**Symptom**: "Error running query" message

**Solutions**:
1. Check your database credentials in profiles.yml
2. Verify target is correctly selected
3. Test connection: `dbt debug`
4. Check the compiled SQL for errors

#### Slow Performance

**Symptom**: UI is laggy or unresponsive

**Solutions**:
1. Disable realtime compilation
2. Reduce query result size via template
3. Close other browser tabs
4. Restart the workbench

### Getting Help

- **Refresh the page**: Reloads dbt context
- **Restart workbench**: `Ctrl+C` and re-run command
- **Check logs**: Look for errors in terminal output

---

## Quick Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│                 dbt-osmosis Workbench Quick Reference          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  START:        dbt-osmosis workbench                           │
│  STOP:         Ctrl+C in terminal                              │
│                                                                │
│  COMPILE:      Ctrl+Enter                                      │
│  RUN QUERY:    Ctrl+Shift+Enter                                │
│  SAVE MODEL:   Click "Save" button                             │
│                                                                │
│  SWITCH MODEL: Sidebar → Models → Select from dropdown         │
│  SWITCH TARGET: Sidebar → Profiles → Click target              │
│                                                                │
│  PROFILE DATA: Run query → Click "Profile Results"             │
│                                                                │
│  REALTIME:     Toggle switch at bottom of editor               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

*User Guide for dbt-osmosis Workbench v1.1.17*
*Last Updated: December 2024*
