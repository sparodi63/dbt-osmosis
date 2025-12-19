# dbt-osmosis Documentation Index

Welcome to the comprehensive documentation for **dbt-osmosis**! This index provides an organized overview of all available documentation resources.

## 📚 Documentation Overview

This documentation set includes:

1. **DBT_OSMOSIS_SUMMARY.md** - Executive summary and high-level overview
2. **DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md** - Detailed feature documentation
3. **DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md** - Deep technical dive into architecture

## 🎯 Quick Start Guide

### What is dbt-osmosis?

**dbt-osmosis** is a CLI tool that enhances dbt development by:
- ✅ Automating schema YAML management
- ✅ Enabling documentation inheritance across data lineage
- ✅ Providing an interactive workbench for SQL development

### Installation

```bash
# Basic installation
pip install dbt-osmosis

# With workbench (Streamlit)
pip install "dbt-osmosis[workbench]"

# With OpenAI support
pip install "dbt-osmosis[openai]"
```

### Quick Example

```bash
# Configure dbt_project.yml
models:
  your_project:
    +dbt-osmosis: "_{model}.yml"

# Run refactor
dbt-osmosis yaml refactor
```

## 📖 Documentation Structure

### 1. [DBT_OSMOSIS_SUMMARY.md](DBT_OSMOSIS_SUMMARY.md)

**For**: Executives, project managers, and new team members

**Covers**:
- Executive summary and value proposition
- High-level architecture overview
- Key features and benefits
- Usage examples
- Integration points (pre-commit, CI/CD, Docker)
- Best practices
- Future directions

**Key Sections**:
- Core Value Proposition
- Architecture Overview
- How It Works
- Usage Examples
- Benefits
- Getting Started
- Best Practices

### 2. [DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md](DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md)

**For**: Data engineers, analysts, and developers

**Covers**:
- Detailed feature descriptions
- Configuration reference
- Usage examples (basic and advanced)
- Plugin system
- Integration with dbt
- Troubleshooting
- Development setup

**Key Sections**:
- Core Features (YAML Management, SQL Operations, Workbench)
- Configuration Reference
- Usage Examples
- Plugin System
- Integration with dbt
- Best Practices
- Troubleshooting
- Development

### 3. [DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md](DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md)

**For**: Developers, maintainers, and contributors

**Covers**:
- Technical architecture
- Key algorithms
- Data flow diagrams
- Performance optimization
- Error handling
- Integration points
- Testing strategy
- Deployment considerations

**Key Sections**:
- Core Architecture (Transformation Pipeline, Configuration System, Documentation Inheritance)
- Key Algorithms (Column Matching, Topological Sorting, YAML Synchronization)
- Data Flow
- Performance Optimization
- Error Handling
- Integration Points
- Testing Strategy
- Deployment Considerations

## 🎯 Feature Quick Reference

### YAML Management Commands

| Command | Purpose | Key Options |
|---------|---------|-------------|
| `yaml refactor` | Complete workflow (organize + document) | `--dry-run`, `--check`, `--auto-apply` |
| `yaml organize` | Restructure YAML files | `--auto-apply` |
| `yaml document` | Inherit documentation | `--force-inherit-descriptions`, `--synthesize` |

### SQL Operations

| Command | Purpose | Example |
|---------|---------|---------|
| `sql run` | Execute SQL | `dbt-osmosis sql run "SELECT * FROM {{ ref('model') }}"` |
| `sql compile` | Compile SQL | `dbt-osmosis sql compile "SELECT * FROM {{ ref('model') }}"` |

### Workbench

| Command | Purpose | Features |
|---------|---------|----------|
| `workbench` | Interactive development | Editor, Query Tester, Data Profiler, Theme Customization |

## 🔧 Configuration Quick Reference

### Basic Configuration

```yaml
models:
  your_project:
    +dbt-osmosis: "_{model}.yml"
    
    staging:
      +dbt-osmosis: "{parent}.yml"
      +dbt-osmosis-options:
        skip-add-columns: true
        sort-by: "alphabetical"
```

### Common Options

| Option | Description | Default |
|--------|-------------|---------|
| `skip-add-columns` | Skip adding missing columns | `false` |
| `skip-add-data-types` | Skip adding data types | `false` |
| `skip-merge-meta` | Skip merging meta from upstream | `false` |
| `skip-add-tags` | Skip inheriting tags | `false` |
| `sort-by` | Column sorting method | `database` |
| `force-inherit-descriptions` | Override existing descriptions | `false` |

## 🚀 Usage Patterns

### Pattern 1: Initial Setup

```bash
# 1. Configure dbt_project.yml
# 2. Run dry run to preview changes
dbt-osmosis yaml refactor --dry-run

# 3. Review and apply
dbt-osmosis yaml refactor --auto-apply
```

### Pattern 2: CI/CD Integration

```yaml
# .github/workflows/ci.yml
- name: Run dbt-osmosis
  run: dbt-osmosis yaml refactor --check
```

### Pattern 3: Pre-commit Hook

```yaml
# .pre-commit-config.yaml
- id: dbt-osmosis
  files: ^models/
  args: [--dry-run, --check]
```

### Pattern 4: Documentation Inheritance

```yaml
# Document at source layer
models:
  - name: stg_customers
    columns:
      - name: customer_id
        description: "Unique identifier for each customer"
      - name: name
        description: "Full name of the customer"

# Documentation automatically flows to downstream models
```

## 🛠️ Troubleshooting Guide

### Common Issues

**Issue: No changes detected**
- Check configuration in `dbt_project.yml`
- Verify models exist in dbt project
- Use `--dry-run` to see what would change

**Issue: Connection errors**
- Verify database credentials in `profiles.yml`
- Use `--catalog-path` to avoid live queries

**Issue: Column matching failures**
- Use plugins for custom matching
- Check for case sensitivity issues
- Review column naming conventions

**Issue: Permission errors**
- Ensure write permissions to models directory
- Check file permissions on existing YAML files

## 📈 Learning Path

### For New Users

1. Start with [DBT_OSMOSIS_SUMMARY.md](DBT_OSMOSIS_SUMMARY.md)
2. Read the "Getting Started" section
3. Try the quick example
4. Explore [DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md](DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md) for details

### For Advanced Users

1. Review [DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md](DBT_OSMOSIS_COMPREHENSIVE_DOCUMENTATION.md)
2. Study configuration options
3. Explore plugin system
4. Read [DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md](DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md) for deep technical details

### For Contributors

1. Read [DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md](DBT_OSMOSIS_TECHNICAL_DOCUMENTATION.md)
2. Study the architecture diagrams
3. Review the testing strategy
4. Check the development setup guide

## 🔗 External Resources

### Official Resources

- **GitHub Repository**: https://github.com/z3z1ma/dbt-osmosis
- **Documentation Site**: https://z3z1ma.github.io/dbt-osmosis/
- **PyPI Package**: https://pypi.org/project/dbt-osmosis/
- **Changelog**: https://github.com/z3z1ma/dbt-osmosis/blob/main/CHANGELOG.md

### Related Tools

- **dbt**: https://docs.getdbt.com/
- **dbt-core**: https://github.com/dbt-labs/dbt-core
- **Streamlit**: https://streamlit.io/
- **Click**: https://click.palletsprojects.com/

## 📞 Support

### Community Support

- GitHub Issues: https://github.com/z3z1ma/dbt-osmosis/issues
- GitHub Discussions: https://github.com/z3z1ma/dbt-osmosis/discussions
- dbt Slack: #dbt-osmosis channel

### Professional Support

For enterprise support or custom development, consider:
- Hiring the maintainers
- Engaging dbt consulting partners
- Custom development contracts

## 🎓 Training Resources

### Video Tutorials

- [dbt-osmosis Overview](https://www.youtube.com/watch?v=example)
- [Configuration Guide](https://www.youtube.com/watch?v=example)
- [Workbench Demo](https://www.youtube.com/watch?v=example)

### Workshops

- dbt Coalesce workshops
- Community meetups
- Online webinars

## 📅 Release Notes

### Latest Release

**Version**: v1.1.5

**Key Features**:
- Enhanced configuration system
- Improved documentation inheritance
- Better error handling
- Performance optimizations

**Breaking Changes**: None

**Migration Guide**: [Migrating from 0.x.x to 1.x.x](https://z3z1ma.github.io/dbt-osmosis/docs/migrating)

## 🔍 Search Documentation

Use your system's search (Ctrl+F / Cmd+F) to find:
- Specific commands
- Configuration options
- Error messages
- Feature descriptions
- Integration examples

## 📝 Contributing

### How to Contribute

1. **Report Issues**: File bugs or feature requests on GitHub
2. **Submit PRs**: Fix bugs or add features
3. **Improve Docs**: Enhance documentation
4. **Spread the Word**: Share with the community

### Contribution Guidelines

- Follow Python best practices
- Add tests for new features
- Update documentation
- Follow the code style guide

## 🎯 Summary

This documentation index provides a comprehensive guide to dbt-osmosis. Whether you're:
- A **new user** looking to get started
- A **data engineer** managing dbt projects
- A **developer** contributing to the project
- A **team lead** evaluating tools

You'll find the information you need in these documents. Happy documenting! 📝

## 📖 Next Steps

1. **Read the summary** to understand the big picture
2. **Try the quick example** to see it in action
3. **Explore the comprehensive docs** for detailed usage
4. **Dive into technical docs** if you need implementation details
5. **Join the community** for support and discussions

----

**Last Updated**: 2024
**Documentation Version**: 1.0
**dbt-osmosis Version**: 1.1.5
