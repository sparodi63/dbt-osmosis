# dbt-osmosis Maintenance Guide

## Table of Contents

1. [Project Health Overview](#1-project-health-overview)
2. [Scheduled Maintenance Tasks](#2-scheduled-maintenance-tasks)
3. [Critical Code Areas](#3-critical-code-areas)
4. [Dependency Management](#4-dependency-management)
5. [Testing Strategy](#5-testing-strategy)
6. [Release Checklist](#6-release-checklist)
7. [Troubleshooting Runbook](#7-troubleshooting-runbook)
8. [Technical Debt Inventory](#8-technical-debt-inventory)
9. [Upgrade Procedures](#9-upgrade-procedures)

---

## 1. Project Health Overview

### 1.1 Current Status

| Metric | Value | Notes |
|--------|-------|-------|
| Version | 1.1.17 | Current release |
| Python Support | 3.10, 3.11, 3.12 | 3.13 not yet supported |
| dbt-core Support | 1.8 - 1.10.x | May need updates for 1.11+ |
| Last Major Update | Fork from z3z1ma/dbt-osmosis | Ubie-oss fork with loosened constraints |
| Lines of Code | ~5,222 | src/ directory |
| Test Coverage | Needs measurement | Run coverage report |

### 1.2 Key Maintainer Responsibilities

1. **Compatibility**: Keep pace with dbt-core releases
2. **Stability**: Ensure YAML operations are safe and reversible
3. **Performance**: Maintain reasonable execution times
4. **Security**: Review LLM integrations and credential handling

---

## 2. Scheduled Maintenance Tasks

### 2.1 Daily (If Active Development)

| Task | Command/Action | Time |
|------|----------------|------|
| Check CI status | Review GitHub Actions | 5 min |
| Review new issues | GitHub Issues | 10 min |
| Monitor dependencies | `pip list --outdated` | 5 min |

### 2.2 Weekly Maintenance

```bash
# 1. Update local development environment
git pull origin main
pip install -e ".[dev]"

# 2. Run full test suite
pytest tests/ -v

# 3. Run linting
ruff check src/
ruff format --check src/

# 4. Check for security vulnerabilities
pip-audit

# 5. Update pre-commit hooks
pre-commit autoupdate
```

### 2.3 Monthly Maintenance

| Task | Description | Priority |
|------|-------------|----------|
| **Dependency Audit** | Review all dependencies for updates | High |
| **dbt-core Testing** | Test with latest dbt-core release | High |
| **Performance Baseline** | Run benchmarks on sample projects | Medium |
| **Documentation Review** | Ensure docs match current code | Medium |
| **Issue Triage** | Categorize and prioritize open issues | Medium |
| **Security Scan** | Run comprehensive security audit | High |

### 2.4 Quarterly Maintenance

| Task | Description |
|------|-------------|
| **Major Version Testing** | Test compatibility with new Python/dbt versions |
| **Architecture Review** | Evaluate technical debt and refactoring needs |
| **Roadmap Update** | Plan features for next quarter |
| **Community Engagement** | Respond to stale issues/PRs |
| **Documentation Overhaul** | Major doc updates if needed |

### 2.5 Annual Maintenance

| Task | Description |
|------|-------------|
| **Python EOL Review** | Drop support for EOL Python versions |
| **dbt-core Major Version** | Major compatibility work for new dbt versions |
| **License Compliance** | Verify all dependencies remain compatible |
| **Contributor Guide Update** | Refresh development documentation |

---

## 3. Critical Code Areas

### 3.1 High-Risk Areas (Require Careful Review)

#### 3.1.1 YAML Buffer Cache (`core/schema/reader.py`, `core/schema/writer.py`)

```python
# Critical global state
_YAML_BUFFER_CACHE: dict[Path, Any] = {}
```

**Risks:**
- Memory leaks in long-running processes
- Stale data if files change externally
- Thread safety issues

**Maintenance Actions:**
- Review all cache access points
- Ensure proper cache invalidation
- Monitor memory usage in tests

#### 3.1.2 dbt-core Integration (`core/config.py`)

**Critical Functions:**
- `create_dbt_project_context()` - Initializes dbt runtime
- `_instantiate_adapter()` - Creates database connection
- `_reload_manifest()` - Updates manifest after changes

**Risks:**
- dbt-core API changes
- Adapter compatibility issues
- Connection leaks

**Maintenance Actions:**
- Test with each dbt-core minor version
- Review dbt-core changelog for breaking changes
- Monitor adapter deprecation warnings

#### 3.1.3 Transform Pipeline (`core/transforms.py`)

**Critical Operations:**
- `inject_missing_columns` - Adds columns from database
- `remove_columns_not_in_database` - Removes orphaned columns
- `inherit_upstream_column_knowledge` - Propagates documentation

**Risks:**
- Data loss if columns removed incorrectly
- Infinite loops in inheritance
- Race conditions in parallel execution

**Maintenance Actions:**
- Extensive testing with varied project structures
- Verify dry-run mode works correctly
- Test with circular dependency detection

#### 3.1.4 LLM Integration (`core/llm.py`)

**Risks:**
- API key exposure in logs
- Cost overruns from excessive calls
- Breaking changes in LLM provider APIs

**Maintenance Actions:**
- Audit logging for sensitive data
- Implement rate limiting
- Pin LLM SDK versions
- Test with mock responses

### 3.2 Medium-Risk Areas

| File | Concern | Review Frequency |
|------|---------|------------------|
| `core/introspection.py` | Adapter-specific column handling | Monthly |
| `core/restructuring.py` | File system operations | Monthly |
| `core/path_management.py` | Path template parsing | Quarterly |
| `core/node_filters.py` | Topological sort correctness | Quarterly |

### 3.3 Low-Risk Areas

| File | Concern | Review Frequency |
|------|---------|------------------|
| `core/logger.py` | Logging configuration | Yearly |
| `core/plugins.py` | Plugin system | Yearly |
| `cli/main.py` | CLI options | When adding features |

---

## 4. Dependency Management

### 4.1 Core Dependencies Status

| Package | Current Constraint | Latest | Action Needed |
|---------|-------------------|--------|---------------|
| dbt-core | >=1.8,<1.11 | Check PyPI | Update upper bound for new releases |
| ruamel.yaml | >=0.17,<0.19 | Check PyPI | Test with new versions |
| click | >7,<9 | 8.x | Stable, monitor only |
| rich | >=10 | Check PyPI | Stable, update freely |
| pluggy | >=1.5.0,<2 | Check PyPI | Stable, monitor only |
| mysql-mimic | >=2.5.7 | Check PyPI | Watch for breaking changes |

### 4.2 Updating Dependencies

```bash
# 1. Check outdated packages
pip list --outdated

# 2. Update in development
pip install -U package_name

# 3. Run tests
pytest tests/ -v

# 4. If tests pass, update pyproject.toml
# Be conservative with upper bounds for core deps

# 5. Create PR with dependency update
```

### 4.3 dbt-core Compatibility Matrix

| dbt-core Version | osmosis Version | Status | Notes |
|------------------|-----------------|--------|-------|
| 1.8.x | 1.1.x | Supported | Minimum supported |
| 1.9.x | 1.1.x | Supported | Tested |
| 1.10.x | 1.1.x | Supported | Latest tested |
| 1.11.x | TBD | Needs testing | Update constraint after testing |

### 4.4 Breaking Change Monitoring

**Watch these dbt-core areas:**
- `dbt.contracts.graph.manifest.Manifest` structure
- `dbt.adapters.base.impl.BaseAdapter` interface
- `dbt.config.runtime.RuntimeConfig` API
- `dbt.parser.*` classes

**Common breaking patterns:**
- Renamed/moved imports
- Changed method signatures
- New required arguments
- Deprecation warnings becoming errors

---

## 5. Testing Strategy

### 5.1 Test Categories

| Category | Location | Purpose | Run Frequency |
|----------|----------|---------|---------------|
| Unit | `tests/core/` | Individual module tests | Every commit |
| Integration | `tests/test_yaml_*.py` | End-to-end YAML operations | Every commit |
| Legacy | `tests/core/test_legacy.py` | Backwards compatibility | Every commit |

### 5.2 Running Tests

```bash
# Full test suite
pytest tests/ -v

# Specific module
pytest tests/core/test_transforms.py -v

# With coverage
pytest tests/ --cov=src/dbt_osmosis --cov-report=html

# Parallel execution (faster)
pytest tests/ -n auto

# Only failed tests from last run
pytest tests/ --lf
```

### 5.3 Test Infrastructure Requirements

- **DuckDB**: Used as test database adapter
- **Sample Project**: Tests may require sample dbt project
- **Mock Objects**: For LLM and external API tests

### 5.4 Adding New Tests

When adding features, ensure:
1. Unit tests for new functions
2. Integration tests for user-facing behavior
3. Edge case coverage (empty inputs, errors)
4. Dry-run verification

### 5.5 Test Fixtures

```python
# Common fixtures to use/maintain
@pytest.fixture
def sample_dbt_project():
    """Provides a minimal dbt project for testing."""

@pytest.fixture
def mock_manifest():
    """Provides a mock dbt manifest."""

@pytest.fixture
def yaml_context():
    """Provides YamlRefactorContext for testing."""
```

---

## 6. Release Checklist

### 6.1 Pre-Release

- [ ] All tests passing on CI
- [ ] No critical issues open
- [ ] CHANGELOG updated
- [ ] Version bumped in `pyproject.toml`
- [ ] Documentation updated
- [ ] Dependency constraints reviewed

### 6.2 Release Process

```bash
# 1. Ensure main branch is up to date
git checkout main
git pull origin main

# 2. Verify tests pass
pytest tests/ -v

# 3. Update version in pyproject.toml
# version = "1.1.18"

# 4. Update CHANGELOG.md

# 5. Commit version bump
git add pyproject.toml CHANGELOG.md
git commit -m "Release v1.1.18"

# 6. Create and push tag
git tag -a v1.1.18 -m "Release v1.1.18"
git push origin main --tags

# 7. Build and publish (if not using CI)
python -m build
twine upload dist/*
```

### 6.3 Post-Release

- [ ] Verify package on PyPI
- [ ] Test installation: `pip install dbt-osmosis==1.1.18`
- [ ] Update GitHub release notes
- [ ] Announce if significant changes

---

## 7. Troubleshooting Runbook

### 7.1 Common Issues

#### Issue: Tests Fail with Import Errors

**Symptoms:**
```
ModuleNotFoundError: No module named 'dbt_osmosis.core.xyz'
```

**Diagnosis:**
1. Check if module was renamed/moved
2. Verify `__init__.py` exports
3. Check circular imports

**Resolution:**
```bash
pip install -e .
python -c "from dbt_osmosis.core import *"
```

#### Issue: YAML Changes Not Applied

**Symptoms:**
- Changes visible in dry-run but not on disk
- Cache seems stale

**Diagnosis:**
1. Check `dry_run` setting
2. Verify file permissions
3. Check cache state

**Resolution:**
```python
# Clear cache manually if needed
from dbt_osmosis.core.schema.reader import _YAML_BUFFER_CACHE
_YAML_BUFFER_CACHE.clear()
```

#### Issue: dbt-core Compatibility Error

**Symptoms:**
```
AttributeError: 'RuntimeConfig' object has no attribute 'xyz'
```

**Diagnosis:**
1. Check dbt-core version
2. Compare with supported versions
3. Review dbt-core changelog

**Resolution:**
1. Pin to supported version: `pip install dbt-core==1.9.0`
2. Or update code for new API

#### Issue: LLM API Errors

**Symptoms:**
```
openai.RateLimitError: Rate limit exceeded
ValueError: LLM returned empty response
```

**Diagnosis:**
1. Check API key validity
2. Verify environment variables
3. Check API quotas

**Resolution:**
```bash
# Test connection
dbt-osmosis test_llm

# Check environment
echo $OPENAI_API_KEY
echo $LLM_PROVIDER
```

### 7.2 Debug Mode

Enable verbose logging:
```bash
dbt-osmosis yaml refactor --log-level DEBUG 2>&1 | tee debug.log
```

### 7.3 Collecting Diagnostics

```bash
# System info
python --version
pip show dbt-osmosis dbt-core ruamel.yaml

# dbt info
dbt --version
dbt debug

# Project info
cat dbt_project.yml | head -20
```

---

## 8. Technical Debt Inventory

### 8.1 Known Technical Debt

| Item | Location | Severity | Effort | Description |
|------|----------|----------|--------|-------------|
| Type hints incomplete | Various | Low | Medium | Some functions lack full typing |
| Test coverage gaps | `tests/` | Medium | High | Need more edge case tests |
| Error messages unclear | Various | Low | Low | Some errors lack context |
| Cache invalidation | `schema/reader.py` | Medium | Medium | No TTL or size limits on cache |
| Azure OpenAI legacy SDK | `llm.py` | Medium | Low | Uses legacy SDK structure |

### 8.2 Refactoring Candidates

| Area | Current State | Proposed Improvement |
|------|---------------|---------------------|
| `osmosis.py` | Re-exports for compatibility | Consider deprecation path |
| Transform pipeline | Functional approach | Could use class-based for state |
| YAML caching | Global dict | Consider LRU cache with TTL |
| Settings lookup | Multiple dict lookups | Could pre-compute at init |

### 8.3 Deprecated Code

| Item | Location | Deprecated Since | Remove By |
|------|----------|------------------|-----------|
| `osmosis.py` re-exports | `core/osmosis.py` | Original design | v2.0 |

---

## 9. Upgrade Procedures

### 9.1 Upgrading dbt-core Support

When a new dbt-core version releases:

1. **Review Changelog**
   ```bash
   # Check dbt-core changelog for breaking changes
   open https://github.com/dbt-labs/dbt-core/blob/main/CHANGELOG.md
   ```

2. **Create Feature Branch**
   ```bash
   git checkout -b feat/dbt-core-1.11-support
   ```

3. **Update Constraint**
   ```toml
   # pyproject.toml
   dependencies = [
       "dbt-core>=1.8,<1.12",  # Expand upper bound
   ]
   ```

4. **Test Installation**
   ```bash
   pip install dbt-core==1.11.0
   pip install -e ".[dev]"
   ```

5. **Run Tests**
   ```bash
   pytest tests/ -v
   ```

6. **Fix Compatibility Issues**
   - Address import changes
   - Update API calls
   - Handle deprecations

7. **Update Documentation**
   - Update compatibility matrix
   - Add migration notes if needed

8. **Create PR**
   ```bash
   git push origin feat/dbt-core-1.11-support
   ```

### 9.2 Upgrading Python Support

When adding Python 3.13 support:

1. **Update pyproject.toml**
   ```toml
   requires-python = ">=3.10,<3.14"
   classifiers = [
       "Programming Language :: Python :: 3.13",
   ]
   ```

2. **Test with New Version**
   ```bash
   pyenv install 3.13.0
   pyenv local 3.13.0
   pip install -e ".[dev]"
   pytest tests/ -v
   ```

3. **Update CI Matrix**
   - Add Python 3.13 to GitHub Actions

4. **Address Deprecations**
   - Check for deprecated stdlib usage
   - Review typing changes

### 9.3 Dropping Old Python Versions

When dropping Python 3.10 support:

1. **Update Constraint**
   ```toml
   requires-python = ">=3.11,<3.14"
   ```

2. **Remove from CI**

3. **Modernize Code**
   - Use new syntax (match statements, etc.)
   - Remove compatibility shims

4. **Update Documentation**

5. **Create Major Version** (if significant changes)

---

## Appendix A: Useful Commands

```bash
# Development setup
pip install -e ".[dev,workbench,openai]"

# Run tests with coverage
pytest tests/ --cov=src/dbt_osmosis --cov-report=term-missing

# Lint and format
ruff check src/ --fix
ruff format src/

# Type checking
pyright src/

# Build package
python -m build

# Check package
twine check dist/*

# Install specific dbt adapter for testing
pip install dbt-duckdb dbt-snowflake dbt-bigquery

# Run specific test pattern
pytest tests/ -k "test_transform" -v
```

## Appendix B: Contact & Resources

| Resource | Location |
|----------|----------|
| Original Repository | https://github.com/z3z1ma/dbt-osmosis |
| Fork Repository | https://github.com/sparodi63/dbt-osmosis |
| dbt-core | https://github.com/dbt-labs/dbt-core |
| ruamel.yaml | https://yaml.readthedocs.io/en/latest/ |
| pluggy | https://pluggy.readthedocs.io/ |

---

*Maintenance Guide for dbt-osmosis v1.1.17*
*Last Updated: December 2024*
