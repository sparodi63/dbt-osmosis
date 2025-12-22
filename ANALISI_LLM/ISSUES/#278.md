# Solution for Issue #278: Empty config with empty tags and meta keys added

## Issue Summary

**Issue**: [#278](https://github.com/z3z1ma/dbt-osmosis/issues/278)
**Title**: Bug: Empty config with empty tags and empty meta keys added
**Severity**: Medium
**Affected Versions**: dbt-osmosis with dbt-core >= 1.9.6

---

## Problem Description

### Symptoms

When running dbt-osmosis commands (particularly `yaml document`), empty configuration blocks are being added to YAML files:

```yaml
# BEFORE (clean)
columns:
  - name: customer_id
    description: "Unique customer identifier"

# AFTER (polluted)
columns:
  - name: customer_id
    description: "Unique customer identifier"
    config:
      meta: {}
      tags: []
```

This happens even when:
- Using `--skip-add-tags` flag
- Using `--skip-merge-meta` flag
- No upstream models have tags or meta defined

### User Impact

- **12+ users affected** (based on GitHub reactions)
- YAML files become cluttered with unnecessary empty blocks
- Git diffs show spurious changes
- Violates principle of least surprise

---

## Root Cause Analysis

### The dbt-core Change

In **dbt-core 1.9.6** (released via [PR #11671](https://github.com/dbt-labs/dbt-core/pull/11671)), the column structure was modified:

**Before 1.9.6:**
```python
class ColumnInfo:
    meta: dict = {}
    tags: list = []
    # meta and tags are top-level properties
```

**After 1.9.6:**
```python
class ColumnInfo:
    config: ColumnConfig  # NEW: contains meta and tags
    meta: dict = {}       # Deprecated, but still present
    tags: list = []       # Deprecated, but still present
```

When `ColumnInfo.to_dict(omit_none=True)` is called, it now returns:

```python
{
    "name": "column_name",
    "description": "...",
    "config": {
        "meta": {},
        "tags": []
    },
    # ... other fields
}
```

The `omit_none=True` parameter only omits `None` values, not empty dicts `{}` or empty lists `[]`.

### The dbt-osmosis Code Gap

In `src/dbt_osmosis/core/sync_operations.py`, the `_sync_doc_section()` function has cleanup logic for empty values:

```python
# Lines 59-68 - Current cleanup logic
if merged.get("description") is None:
    merged.pop("description", None)
if merged.get("tags", []) == []:
    merged.pop("tags", None)
if merged.get("meta", {}) == {}:
    merged.pop("meta", None)

for k in set(merged.keys()) - {"name", "description", "tags", "meta"}:
    if merged[k] in (None, [], {}):
        merged.pop(k)
```

**Problem**: This code:
1. Only cleans top-level `tags` and `meta`
2. Does NOT clean the new `config` block
3. The generic cleanup (lines 66-68) excludes `tags` and `meta` from the check
4. The `config` dict itself is not empty (it contains `{"meta": {}, "tags": []}`), so it's not caught

---

## Proposed Solution

### Overview

Implement comprehensive cleanup of empty nested structures, specifically targeting the new `config` block introduced in dbt-core 1.9.6.

### Solution Strategy

1. **Add specific `config` cleanup**: Remove empty `config.meta` and `config.tags`
2. **Remove empty `config` block**: If `config` becomes empty after cleanup, remove it entirely
3. **Maintain backwards compatibility**: Keep existing top-level cleanup for older dbt-core versions
4. **Add recursive empty cleanup**: Clean any nested empty structures

---

## Detailed Implementation Plan

### Step 1: Modify `_sync_doc_section()` in `sync_operations.py`

**File**: `src/dbt_osmosis/core/sync_operations.py`
**Function**: `_sync_doc_section()`
**Lines**: 59-68

#### Current Code (Lines 59-68):

```python
if merged.get("description") is None:
    merged.pop("description", None)
if merged.get("tags", []) == []:
    merged.pop("tags", None)
if merged.get("meta", {}) == {}:
    merged.pop("meta", None)

for k in set(merged.keys()) - {"name", "description", "tags", "meta"}:
    if merged[k] in (None, [], {}):
        merged.pop(k)
```

#### Proposed Code:

```python
# Clean up None description
if merged.get("description") is None:
    merged.pop("description", None)

# Clean up empty top-level tags (backwards compatibility for dbt-core < 1.9.6)
if merged.get("tags", []) == []:
    merged.pop("tags", None)

# Clean up empty top-level meta (backwards compatibility for dbt-core < 1.9.6)
if merged.get("meta", {}) == {}:
    merged.pop("meta", None)

# Clean up config block (dbt-core >= 1.9.6)
# The config block may contain empty meta and tags that need removal
if "config" in merged:
    config = merged["config"]
    if isinstance(config, dict):
        # Remove empty tags from config
        if config.get("tags", []) == []:
            config.pop("tags", None)
        # Remove empty meta from config
        if config.get("meta", {}) == {}:
            config.pop("meta", None)
        # If config is now empty or only contains empty values, remove it entirely
        if not config or all(v in (None, [], {}) for v in config.values()):
            merged.pop("config", None)

# Clean up any other keys with empty values
for k in list(merged.keys()):
    if k in ("name",):  # Never remove name
        continue
    if merged[k] in (None, [], {}):
        merged.pop(k, None)
```

### Step 2: Add Helper Function for Recursive Cleanup (Optional Enhancement)

For a more robust solution, add a helper function to handle nested empty structures:

**Add before `_sync_doc_section()`**:

```python
def _clean_empty_values(d: dict[str, t.Any], preserve_keys: set[str] | None = None) -> dict[str, t.Any]:
    """
    Recursively remove empty values (None, [], {}) from a dictionary.

    Args:
        d: Dictionary to clean
        preserve_keys: Set of keys that should never be removed (e.g., {"name"})

    Returns:
        Cleaned dictionary with empty values removed
    """
    if preserve_keys is None:
        preserve_keys = set()

    cleaned = {}
    for k, v in d.items():
        # Always preserve specified keys
        if k in preserve_keys:
            cleaned[k] = v
            continue

        # Recursively clean nested dicts
        if isinstance(v, dict):
            nested = _clean_empty_values(v)
            if nested:  # Only include if not empty after cleaning
                cleaned[k] = nested
        # Skip empty lists
        elif isinstance(v, list) and len(v) == 0:
            continue
        # Skip None
        elif v is None:
            continue
        # Keep everything else
        else:
            cleaned[k] = v

    return cleaned
```

Then simplify `_sync_doc_section()`:

```python
# After building merged dict...
merged = _clean_empty_values(merged, preserve_keys={"name"})
incoming_columns.append(merged)
```

### Step 3: Apply Same Fix to `inheritance.py`

**File**: `src/dbt_osmosis/core/inheritance.py`
**Function**: `_build_column_knowledge_graph()`
**Line**: 135

The same issue can occur when building the column knowledge graph:

```python
graph_edge = incoming.to_dict(omit_none=True)
```

Add cleanup after this line:

```python
graph_edge = incoming.to_dict(omit_none=True)

# Clean up empty config block from dbt-core 1.9.6+
if "config" in graph_edge:
    config = graph_edge["config"]
    if isinstance(config, dict):
        if config.get("tags", []) == []:
            config.pop("tags", None)
        if config.get("meta", {}) == {}:
            config.pop("meta", None)
        if not config or all(v in (None, [], {}) for v in config.values()):
            graph_edge.pop("config", None)
```

---

## Implementation Checklist

### Code Changes

- [ ] **sync_operations.py**: Update `_sync_doc_section()` to handle `config` block
- [ ] **sync_operations.py**: Add `_clean_empty_values()` helper (optional)
- [ ] **inheritance.py**: Add cleanup after `to_dict()` call in `_build_column_knowledge_graph()`

### Testing

- [ ] **Unit test**: Test with dbt-core 1.9.5 (pre-change behavior)
- [ ] **Unit test**: Test with dbt-core 1.9.6+ (config block present)
- [ ] **Unit test**: Test that empty config is removed
- [ ] **Unit test**: Test that non-empty config is preserved
- [ ] **Integration test**: Run `yaml document` and verify no empty blocks
- [ ] **Integration test**: Test with `--skip-add-tags --skip-merge-meta`

### Test Cases

```python
# test_sync_operations.py

def test_empty_config_block_removed():
    """Config block with empty meta and tags should be removed."""
    column_dict = {
        "name": "test_col",
        "description": "Test",
        "config": {
            "meta": {},
            "tags": []
        }
    }
    # After cleanup
    assert "config" not in cleaned_column

def test_nonempty_config_preserved():
    """Config block with actual values should be preserved."""
    column_dict = {
        "name": "test_col",
        "config": {
            "meta": {"owner": "team_a"},
            "tags": ["pii"]
        }
    }
    # After cleanup
    assert "config" in cleaned_column
    assert cleaned_column["config"]["meta"] == {"owner": "team_a"}

def test_partial_empty_config_cleaned():
    """Config with some empty values should have only empty parts removed."""
    column_dict = {
        "name": "test_col",
        "config": {
            "meta": {"owner": "team_a"},
            "tags": []  # Empty, should be removed
        }
    }
    # After cleanup
    assert "config" in cleaned_column
    assert "tags" not in cleaned_column["config"]
    assert cleaned_column["config"]["meta"] == {"owner": "team_a"}

def test_backwards_compatibility_pre_196():
    """Top-level meta and tags should still be cleaned (dbt-core < 1.9.6)."""
    column_dict = {
        "name": "test_col",
        "meta": {},
        "tags": []
    }
    # After cleanup
    assert "meta" not in cleaned_column
    assert "tags" not in cleaned_column
```

---

## Complete Patch

### File: `src/dbt_osmosis/core/sync_operations.py`

```diff
--- a/src/dbt_osmosis/core/sync_operations.py
+++ b/src/dbt_osmosis/core/sync_operations.py
@@ -14,6 +14,42 @@ __all__ = [
 ]


+def _clean_empty_values(
+    d: dict[str, t.Any],
+    preserve_keys: t.Optional[set[str]] = None
+) -> dict[str, t.Any]:
+    """
+    Recursively remove empty values (None, [], {}) from a dictionary.
+
+    Args:
+        d: Dictionary to clean
+        preserve_keys: Set of keys that should never be removed (e.g., {"name"})
+
+    Returns:
+        Cleaned dictionary with empty values removed
+    """
+    if preserve_keys is None:
+        preserve_keys = set()
+
+    cleaned = {}
+    for k, v in d.items():
+        # Always preserve specified keys
+        if k in preserve_keys:
+            cleaned[k] = v
+            continue
+
+        # Recursively clean nested dicts
+        if isinstance(v, dict):
+            nested = _clean_empty_values(v)
+            if nested:  # Only include if not empty after cleaning
+                cleaned[k] = nested
+        elif isinstance(v, list) and len(v) == 0:
+            continue  # Skip empty lists
+        elif v is None:
+            continue  # Skip None
+        else:
+            cleaned[k] = v
+
+    return cleaned
+
+
 def _sync_doc_section(context: t.Any, node: ResultNode, doc_section: dict[str, t.Any]) -> None:
     """Helper function that overwrites 'doc_section' with data from 'node'.

@@ -56,17 +92,8 @@ def _sync_doc_section(context: t.Any, node: ResultNode, doc_section: dict[str, t
             continue
         merged[k] = v

-    if merged.get("description") is None:
-        merged.pop("description", None)
-    if merged.get("tags", []) == []:
-        merged.pop("tags", None)
-    if merged.get("meta", {}) == {}:
-        merged.pop("meta", None)
-
-    for k in set(merged.keys()) - {"name", "description", "tags", "meta"}:
-        if merged[k] in (None, [], {}):
-            merged.pop(k)
-
+    # Clean all empty values recursively, including nested config block
+    merged = _clean_empty_values(merged, preserve_keys={"name"})
+
     if _get_setting_for_node(
         "output-to-lower", node, name, fallback=context.settings.output_to_lower
     ):
```

### File: `src/dbt_osmosis/core/inheritance.py`

```diff
--- a/src/dbt_osmosis/core/inheritance.py
+++ b/src/dbt_osmosis/core/inheritance.py
@@ -132,6 +132,18 @@ def _build_column_knowledge_graph(context: t.Any, node: ResultNode) -> dict[str,
                 else:
                     continue
                 graph_edge = incoming.to_dict(omit_none=True)
+
+                # Clean up empty config block from dbt-core 1.9.6+
+                if "config" in graph_edge:
+                    config = graph_edge["config"]
+                    if isinstance(config, dict):
+                        if config.get("tags", []) == []:
+                            config.pop("tags", None)
+                        if config.get("meta", {}) == {}:
+                            config.pop("meta", None)
+                        # Remove config entirely if empty
+                        if not config or all(v in (None, [], {}) for v in config.values()):
+                            graph_edge.pop("config", None)

                 from dbt_osmosis.core.introspection import _get_setting_for_node
```

---

## Alternative Solutions Considered

### Option A: Patch at YAML Write Level

**Approach**: Clean empty values in `schema/writer.py` before writing.

**Pros**: Single point of cleanup
**Cons**: Masks the problem instead of fixing at source; may have unintended effects

**Verdict**: Not recommended

### Option B: Filter in `to_dict()` Wrapper

**Approach**: Create wrapper function for `ColumnInfo.to_dict()`

**Pros**: Centralized fix
**Cons**: Requires finding all call sites; adds indirection

**Verdict**: Could work, but inline fix is simpler

### Option C: Monkey-patch dbt-core

**Approach**: Override `ColumnInfo.to_dict()` behavior

**Pros**: Fixes at root cause
**Cons**: Fragile; breaks with dbt-core updates; not maintainable

**Verdict**: Not recommended

---

## Rollout Plan

1. **Implement fix** in feature branch
2. **Run existing tests** to ensure no regressions
3. **Add new tests** for empty config cleanup
4. **Manual testing** with dbt-core 1.9.5 and 1.9.8+
5. **Create PR** with clear description
6. **Release** as patch version (e.g., 1.1.18)

---

## Verification Steps

After implementing the fix, verify with:

```bash
# 1. Install latest dbt-core
pip install dbt-core>=1.9.6

# 2. Run document command
dbt-osmosis yaml document models/ --skip-add-tags --skip-merge-meta

# 3. Check YAML output - should NOT contain:
#    config:
#      meta: {}
#      tags: []

# 4. Verify non-empty config is preserved
# If a column has actual meta/tags, they should remain
```

---

## Related Issues

- **#281**: Draft PR attempting similar fix (review for overlap)
- **#219**: `--force-inherit-descriptions` + `--use-unrendered-descriptions` (may share code path)
- **#266**: Variables rendered in policy tags (related to config handling)

---

## References

- [dbt-core PR #11671](https://github.com/dbt-labs/dbt-core/pull/11671) - The change that introduced `config` on columns
- [dbt-core Issue #11651](https://github.com/dbt-labs/dbt-core/issues/11651) - Original issue about meta/tags as configs
- [dbt-osmosis Issue #278](https://github.com/z3z1ma/dbt-osmosis/issues/278) - This issue

---

*Solution document for dbt-osmosis Issue #278*
*Prepared: December 2024*
