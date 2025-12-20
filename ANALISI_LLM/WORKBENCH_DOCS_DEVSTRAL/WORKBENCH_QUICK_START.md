# dbt-osmosis Workbench - Quick Start Guide

## 🚀 Quick Start

### 1. Install the Workbench

```bash
pip install "dbt-osmosis[workbench]"
```

### 2. Run the Workbench

```bash
cd /path/to/your/dbt/project
dbt-osmosis workbench
```

The workbench will open in your browser at `http://localhost:8501`

### 3. Start Coding!

- **Select a model** from the dropdown (or use "SCRATCH" for temporary queries)
- **Edit SQL** in the editor panel
- **Compile** with Ctrl+Enter
- **Execute** with Ctrl+Shift+Enter
- **Save** with Ctrl+S (or click the save button)

## 🎯 Key Features at a Glance

### 📝 Editor
- Real-time SQL/Jinja editing
- Syntax highlighting
- Side-by-side compilation view
- Load and save dbt models

### 🔍 Preview
- Execute queries against your database
- View results in a table
- See execution status and errors

### 📊 Profiler
- Generate comprehensive data profiles
- Statistical analysis
- Visualizations and charts

### 💁 Profiles
- Switch between different dbt targets
- Compile and execute against different environments

## 🎨 Interface Layout

```
+---------------------------------------------------+
|  Editor (SQL)  |  Renderer (Compiled SQL)         |
+----------------+-----------------------------------+
|  Preview (Results)  |  Feed (Hacker News)             |
+---------------------+-------------------------------+
|  Profiler (Stats)   |                               |
+---------------------+-------------------------------+
|  Sidebar: Models, Profiles, Query Template      |
+---------------------------------------------------+
```

## ⌨️ Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| Ctrl+Enter | Compile SQL |
| Ctrl+Shift+Enter | Execute Query |
| Ctrl+S | Save Model |
| Command+S (Mac) | Save Model |
| Command+Shift+S (Mac) | Execute Query |

## 💡 Tips

1. **Use SCRATCH mode** for testing without affecting real models
2. **Refresh the page** after changing dbt project files
3. **Test with different targets** to verify cross-environment compatibility
4. **Use LIMIT** in queries to test with smaller datasets
5. **Profile your data** to understand it better before transformation

## 🐛 Troubleshooting

### Workbench won't start?
```bash
pip install "dbt-osmosis[workbench]"
```

### Compilation errors?
- Check your dbt project can be parsed
- Verify all dependencies are installed
- Ensure referenced models exist

### Execution errors?
- Test your database connection with `dbt debug`
- Verify profile credentials are correct
- Check database is accessible

## 📚 Learn More

- [Full Documentation](https://z3z1ma.github.io/dbt-osmosis/)
- [Live Demo](https://dbt-osmosis-playground.streamlit.app/)
- [GitHub Repository](https://github.com/z3z1ma/dbt-osmosis)

## 🎓 Example Workflow

1. **Open workbench**: `dbt-osmosis workbench`
2. **Select model**: Choose `stg_customers` from dropdown
3. **Edit SQL**: Modify the SELECT statement
4. **Compile**: Ctrl+Enter to see compiled SQL
5. **Test**: Ctrl+Shift+Enter to run against dev target
6. **Profile**: Click "Run Profile" to analyze data
7. **Save**: Ctrl+S to persist changes
8. **Test other targets**: Switch to prod target and re-run

Enjoy your dbt development! 🌊
