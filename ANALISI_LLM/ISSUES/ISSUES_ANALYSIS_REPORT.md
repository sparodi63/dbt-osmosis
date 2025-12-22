# dbt-osmosis Open Issues Analysis Report

**Repository**: https://github.com/z3z1ma/dbt-osmosis
**Analysis Date**: December 2024
**Total Open Issues**: 18

---

## Executive Summary

The dbt-osmosis repository has **18 open issues** spanning from June 2024 to November 2025. The issues reveal several key themes:

| Category | Count | Priority |
|----------|-------|----------|
| Documentation Inheritance Bugs | 5 | High |
| YAML/Config Handling | 4 | Medium |
| Feature Requests | 4 | Medium |
| External Package Compatibility | 2 | High |
| Architecture/Refactoring | 2 | Low |
| Whitespace/Formatting | 1 | Low |

**Key Finding**: The most critical issues relate to documentation inheritance edge cases and compatibility with external dbt packages. These affect core functionality and should be prioritized.

---

## Quick Summary Table

| # | Title | Category | Severity | Effort |
|---|-------|----------|----------|--------|
| 289 | Quoted columns with spaces not inheriting | Bug | Medium | Medium |
| 287 | Restore `osmosis_keep_description` option | Feature | High | Medium |
| 286 | `--add-progenitor-to-meta` + `--skip-merge-meta` conflict | Bug | Medium | Low |
| 283 | FileNotFoundError with dbt packages (elementary) | Bug | **Critical** | Medium |
| 279 | YAML filename based on nested directories | Question | Low | Low |
| 278 | Empty config/tags/meta being added | Bug | Medium | Low |
| 277 | Custom LLM prompts | Feature | Low | Medium |
| 276 | End-of-line whitespace in descriptions | Bug | Low | Low |
| 274 | `vars` configuration timing issue | Bug | Medium | Low |
| 273 | Use dbt-core-interface | Architecture | Low | High |
| 272 | Dynamic source names create duplicates | Bug | **High** | Medium |
| 266 | Don't render variables in policy tags | Bug | Medium | Medium |
| 231 | CLI missing `include_external` option | Feature | Medium | Low |
| 219 | `--force-inherit` + `--use-unrendered` broken | Bug | **High** | Medium |
| 217 | New tables not added to existing sources | Bug | Medium | Medium |
| 216 | Macros inheritance instead of rendering | Feature | Medium | Medium |
| 172 | Improve `osmosis_progenitor` accuracy | Enhancement | Low | Medium |
| 167 | dbt Cloud support | Feature | Medium | High |

---

## Detailed Issue Analysis

### Category 1: Documentation Inheritance Issues (5 issues)

These issues affect the core value proposition of dbt-osmosis.

#### #287 - Restore `osmosis_keep_description` [HIGH PRIORITY]
**Problem**: The `--force-inherit-descriptions` flag overwrites custom downstream descriptions. Users who add context-specific descriptions (e.g., "ID of the **latest** shop") lose them when inheritance runs.

**Impact**: Data teams cannot maintain both inheritance AND custom descriptions.

**Suggested Fix**:
- Restore deprecated `osmosis_keep_description` meta field
- Or add `--preserve-custom-descriptions` flag
- Allow per-column override via meta

**Files Affected**: `core/inheritance.py`, `core/transforms.py`

---

#### #219 - `--force-inherit` + `--use-unrendered` broken [HIGH PRIORITY]
**Problem**: When using both flags together, Jinja `{{ doc(...) }}` blocks are rendered even when no upstream column is found.

**Expected**: Preserve `{{ doc("my_desc") }}` syntax
**Actual**: Renders to `"my dummy description"`

**Impact**: 3 thumbs up - multiple users affected

**Root Cause**: Logic in `_build_column_knowledge_graph()` may be rendering descriptions before the "no upstream" check.

**Files Affected**: `core/inheritance.py:104-116`

---

#### #289 - Quoted columns with spaces not inheriting
**Problem**: Columns with spaces in names (e.g., `"Customer Name"`) don't inherit documentation properly.

**Root Cause**: `normalize_column_name()` may strip quotes incorrectly.

**Files Affected**: `core/introspection.py:55-62`

---

#### #216 - Macros should inherit, not render
**Problem**: Users want `{{ doc(...) }}` macros to be inherited as-is, not resolved.

**Related To**: #219

**Files Affected**: `core/inheritance.py`

---

#### #172 - Improve `osmosis_progenitor` accuracy [ENHANCEMENT]
**Problem**: When multiple ancestors have the same column at equal distance, selection may be arbitrary.

**Suggested Fix**: Add deterministic selection (alphabetical, or prefer sources over models).

**Files Affected**: `core/inheritance.py:90-200`

---

### Category 2: External Package Compatibility (2 issues)

#### #283 - FileNotFoundError with dbt packages [CRITICAL]
**Problem**: dbt-osmosis crashes when project includes packages like `elementary`. It searches for package files in source directory instead of `target/compiled/` or `dbt_packages/`.

**Error**: `FileNotFoundError` when processing package models

**Workaround Exists**: Community member provided patch for `_is_file_match()` to check `package_name` attribute.

**Impact**: Blocks any project using external packages

**Files Affected**: `core/node_filters.py:35-56`

**Suggested Fix**:
```python
# In _is_file_match()
if node.package_name != context.project.runtime_cfg.project_name:
    # Construct path using dbt_packages/
    node_path = Path(root, "dbt_packages", node.package_name, node.original_file_path)
```

---

#### #231 - CLI missing `include_external` option
**Problem**: CLI doesn't expose `include_external` parameter that exists in code.

**Related To**: #283

**Fix**: Add `--include-external` flag to CLI commands

**Files Affected**: `cli/main.py`

---

### Category 3: YAML/Configuration Handling (4 issues)

#### #272 - Dynamic source names create duplicates [HIGH PRIORITY]
**Problem**: Sources defined with Jinja variables (e.g., `{{ var("source")[target.name] }}`) create duplicate entries because string comparison fails.

**Root Cause**: `sync_operations.py:148` compares raw YAML string vs resolved node value.

**Impact**: YAML files get polluted with duplicate source definitions.

**Suggested Fix**: Compare against `unrendered_config` or use source identifier matching.

**Files Affected**: `core/sync_operations.py:146-156`

---

#### #278 - Empty config/tags/meta being added
**Problem**: Tool adds empty `config: {}`, `tags: []`, `meta: {}` to YAML entries.

**Expected**: Omit empty keys entirely

**Files Affected**: `core/sync_operations.py:60-68`

**Fix**: Already has cleanup logic, may need stricter filtering.

---

#### #274 - `vars` configuration timing issue
**Problem**: Variables need to be set before `create_dbt_project_context()` is called.

**Root Cause**: `cli/main.py:317-318` sets vars AFTER context creation.

**Fix**: Move vars parsing before context creation.

---

#### #266 - Variables rendered in policy tags
**Problem**: Policy tags containing Jinja variables are being rendered during document/refactor operations.

**Expected**: Keep `{{ var("policy") }}` as-is
**Actual**: Renders to actual policy value

**Related To**: #219, #216

**Files Affected**: `core/inheritance.py`

---

### Category 4: Feature Requests (4 issues)

#### #277 - Custom LLM prompts
**Request**: Allow users to customize prompts for LLM documentation generation.

**Use Case**: Company-specific terminology, documentation standards.

**Suggested Implementation**:
- Add `--llm-prompt-file` CLI option
- Or read from `dbt_project.yml` vars

**Files Affected**: `core/llm.py:122-194`

---

#### #167 - dbt Cloud support [MEDIUM PRIORITY]
**Request**: Support dbt Cloud without requiring local dbt-core setup.

**Community Interest**: 3 heart reactions

**Maintainer Response**: Would need to fetch `catalog.json` from cloud API.

**Workaround**: Use `--catalog-path` with manually downloaded catalog.

**Effort**: High - requires new authentication flow

---

#### #279 - YAML filename based on nested directories
**Type**: Question/Documentation

**Issue**: User unclear how to configure path templates for nested structures.

**Resolution**: Improve documentation for `+dbt-osmosis` path templates.

---

#### #286 - `--add-progenitor-to-meta` + `--skip-merge-meta` conflict
**Problem**: These flags have conflicting behavior.

**Expected**: Add progenitor but skip other meta merging
**Actual**: Progenitor not added when skip-merge-meta is set

**Files Affected**: `core/inheritance.py:139-147`

---

### Category 5: Architecture/Refactoring (2 issues)

#### #273 - Use dbt-core-interface
**Proposal**: Adopt `dbt-core-interface` package for dbt integration.

**Benefit**: More stable interface as dbt-core evolves

**Status**: Assigned to maintainer (z3z1ma)

**Effort**: High - significant refactoring

---

#### #217 - New tables not added to existing source YAML
**Problem**: When a source YAML already exists, new tables aren't appended.

**Root Cause**: Source bootstrap logic only runs for missing sources.

**Files Affected**: `core/path_management.py:168-250`

---

### Category 6: Formatting Issues (1 issue)

#### #276 - End-of-line whitespace in multi-line descriptions
**Problem**: Tool adds trailing spaces to multi-line description fields.

**Files Affected**: `core/schema/parser.py` or ruamel.yaml configuration

**Fix**: Configure YAML dumper to strip trailing whitespace.

---

## Priority Matrix

### Must Fix (Critical/High Impact)

| Issue | Impact | Effort | Recommendation |
|-------|--------|--------|----------------|
| #283 | Blocks package users | Medium | Fix in next release |
| #272 | Data corruption | Medium | Fix in next release |
| #219 | Core feature broken | Medium | Fix in next release |
| #287 | Feature regression | Medium | Consider restoring |

### Should Fix (Medium Impact)

| Issue | Impact | Effort | Recommendation |
|-------|--------|--------|----------------|
| #274 | Config bug | Low | Quick fix |
| #278 | YAML pollution | Low | Quick fix |
| #286 | Flag conflict | Low | Quick fix |
| #266 | Policy tag rendering | Medium | Fix with #219 |
| #231 | Missing CLI option | Low | Quick fix |
| #217 | Incomplete sources | Medium | Improve logic |

### Nice to Have (Low Impact / Enhancements)

| Issue | Impact | Effort | Recommendation |
|-------|--------|--------|----------------|
| #277 | LLM customization | Medium | v2 feature |
| #167 | dbt Cloud | High | Community PR welcome |
| #172 | Progenitor accuracy | Medium | Enhancement |
| #273 | Architecture | High | Long-term roadmap |
| #289 | Edge case | Medium | Investigate |
| #276 | Formatting | Low | Low priority |
| #279 | Documentation | Low | Update docs |
| #216 | Macro inheritance | Medium | Fix with #219 |

---

## Recommended Action Plan

### Phase 1: Critical Fixes (Immediate)

1. **Fix #283**: External package path resolution
   - Modify `_is_file_match()` to handle `dbt_packages/` paths
   - Add tests for elementary, dbt_utils packages

2. **Fix #272**: Dynamic source name matching
   - Compare against unrendered source names
   - Add tests for Jinja-templated sources

3. **Fix #219 + #266 + #216**: Unrendered description handling
   - Audit all places where descriptions are rendered
   - Add `use_unrendered` flag propagation

### Phase 2: Bug Fixes (Next Release)

4. **Fix #274**: Move vars parsing before context creation
5. **Fix #278**: Stricter empty value filtering in sync
6. **Fix #286**: Separate progenitor logic from meta merge
7. **Fix #231**: Add `--include-external` CLI flag
8. **Fix #287**: Restore `osmosis_keep_description` or equivalent

### Phase 3: Enhancements (Future)

9. **Add #277**: Custom LLM prompts
10. **Improve #172**: Deterministic progenitor selection
11. **Consider #273**: dbt-core-interface adoption
12. **Community**: dbt Cloud support (#167)

---

## Repository Health Observations

### Positive Signs
- Active issue reporting indicates user engagement
- Maintainer has responded to several issues
- Community members contributing workarounds
- Clear versioning and releases

### Areas for Improvement
- No issue labels/triage system
- Some issues lack assignees
- No project board for tracking
- Several issues 6+ months old
- Test coverage for edge cases may be lacking

### Recommendations for Maintainers

1. **Implement issue triage**: Add labels (bug, enhancement, question)
2. **Create project board**: Track progress on fixes
3. **Add issue templates**: Ensure reproduction steps included
4. **Improve test coverage**: Add tests for reported edge cases
5. **Document workarounds**: Add known issues to docs
6. **Regular releases**: Address critical bugs promptly

---

## Appendix: Issue Timeline

```
Jun 2024  ████ #167 (dbt Cloud)
...
Feb 2025  ████ #219 (force-inherit + unrendered)
...
Jun 2025  ████ #266 (policy tags)
Jul 2025  ████ #272, #273, #274
Aug 2025  ████ #276, #277, #278
Sep 2025  ████ #279, #283
Oct 2025  ████ #286, #287
Nov 2025  ████ #289
```

---

*Report generated for dbt-osmosis issue analysis*
*Repository: z3z1ma/dbt-osmosis*
