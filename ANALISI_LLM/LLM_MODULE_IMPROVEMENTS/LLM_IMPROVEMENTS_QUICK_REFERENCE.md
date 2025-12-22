# LLM Module Improvements - Quick Reference Guide

## At a Glance

### 📋 Current Issues
- **No retry logic** - Crashes on API failures
- **No client caching** - Creates new client per call
- **Hardcoded prompts** - No customization (Issue #277)
- **Poor error handling** - Generic ValueError
- **No rate limiting** - Can exhaust quotas
- **No logging** - No visibility
- **Legacy Azure SDK** - Deprecated patterns
- **No tests** - Untested code

### ✅ Quick Wins (1-2 hours each)
1. **Custom exceptions** - Better error handling
2. **Retry logic** - Prevent crashes
3. **Custom prompts** - Address Issue #277
4. **Logging** - Debugging visibility
5. **JSON parsing** - More robust extraction

### 📅 Implementation Order

#### Phase 1: Critical Fixes (Immediate)
1. ✅ Custom exceptions
2. ✅ Retry with backoff
3. ✅ Custom prompts (Issue #277)
4. ✅ Logging
5. ✅ JSON parsing improvements

#### Phase 2: Code Quality (Short-term)
1. 📅 Client caching
2. 📅 Modernize Azure client
3. 📅 Provider refactoring
4. 📅 Type hints
5. 📅 Input validation

#### Phase 3: New Features (Long-term)
1. 🔜 Usage tracking
2. 🔜 Timeout handling
3. 🔜 Rate limiting
4. 🔜 JSON mode
5. 🔜 Testing infrastructure

## Implementation Checklist

### Phase 1: Critical Fixes
- [ ] Add custom exception classes (exceptions.py)
- [ ] Implement retry_with_backoff decorator (retry.py)
- [ ] Add client caching with @lru_cache
- [ ] Implement custom prompt loading (prompts.py)
- [ ] Add logging for requests/responses
- [ ] Improve JSON extraction robustness

### Phase 2: Code Quality
- [ ] Modernize Azure OpenAI client
- [ ] Refactor provider configuration
- [ ] Add comprehensive type hints
- [ ] Add input validation

### Phase 3: New Features
- [ ] Implement usage tracking
- [ ] Add timeout configuration
- [ ] Implement rate limiting
- [ ] Add JSON mode support
- [ ] Create MockLLMClient
- [ ] Add unit tests

## New Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `OSMOSIS_LLM_PROMPT_FILE` | None | Custom prompts YAML file |
| `OSMOSIS_LLM_TIMEOUT` | 120 | Request timeout (seconds) |
| `OSMOSIS_LLM_RATE_LIMIT` | 60 | Max requests per minute |
| `OSMOSIS_LLM_MAX_RETRIES` | 3 | Retry attempts |
| `OSMOSIS_LLM_RETRY_DELAY` | 1.0 | Base retry delay (seconds) |
| `OSMOSIS_LLM_TRACK_USAGE` | false | Enable usage tracking |

## New CLI Options

```bash
# Custom prompts
dbt-osmosis yaml document --synthesize --llm-prompt-file prompts.yaml

# Show usage stats
dbt-osmosis yaml document --synthesize --llm-stats

# Set timeout
dbt-osmosis yaml document --synthesize --llm-timeout 60
```

## Code Examples

### Custom Exceptions
```python
class LLMError(Exception):
    """Base exception for LLM operations."""
    pass

class LLMConfigError(LLMError):
    """Raised when LLM configuration is invalid."""
    pass
```

### Retry Decorator
```python
@retry_with_backoff(max_retries=3, base_delay=1.0)
def generate_model_spec_as_json(...):
    # Function implementation
    ...
```

### Custom Prompts
```yaml
# prompts.yaml
column_system: |
  You are a data documentation specialist.
  Follow our standards:
  - Use business terminology
  - Be concise
```

### Usage Tracking
```python
usage = get_usage_stats()
print(usage.summary())
# Output: "LLM Usage: 10 requests, 5000 tokens, ~$0.0750 USD"
```

## Migration Guide

### For Existing Users
- ✅ No breaking changes
- ✅ Existing env vars still work
- ✅ New features are opt-in
- 📅 Azure users: Consider updating SDK (optional)

### Deprecation Timeline
- Legacy Azure SDK: Deprecated v1.2.0, removed v2.0.0
- Direct openai module usage: Deprecated v1.2.0, removed v2.0.0

## Files to Create/Modify

### New Files
- `src/dbt_osmosis/core/llm/exceptions.py` - Custom exceptions
- `src/dbt_osmosis/core/llm/retry.py` - Retry logic
- `src/dbt_osmosis/core/llm/prompts.py` - Prompt templates
- `src/dbt_osmosis/core/llm/usage.py` - Usage tracking
- `src/dbt_osmosis/core/llm/parsing.py` - JSON parsing
- `tests/core/test_llm.py` - Unit tests

### Modified Files
- `src/dbt_osmosis/core/llm.py` - Add new features
- `src/dbt_osmosis/cli/main.py` - Add CLI options

## Success Metrics

- ✅ **Reliability**: % reduction in API failures
- ✅ **Adoption**: % of users using custom prompts
- ✅ **Performance**: % reduction in API calls
- ✅ **Cost**: % reduction in LLM costs
- ✅ **Quality**: % reduction in documentation errors

## Resources

- **Full plan**: LLM_IMPROVEMENTS.md
- **Summary**: LLM_IMPROVEMENTS_SUMMARY.md
- **Quick reference**: LLM_IMPROVEMENTS_QUICK_REFERENCE.md
- **Issue #277**: Custom prompt support

## Need Help?

Start with **Phase 1** - these are the most impactful changes with the lowest risk. Each improvement is self-contained and can be implemented independently.
