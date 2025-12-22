# LLM Module Improvements - Executive Summary

## Overview

The `src/dbt_osmosis/core/llm.py` module provides LLM-powered documentation synthesis for dbt models and columns. This document summarizes the key improvements needed and provides a prioritized implementation plan.

## Current State

- **489 lines** of code
- Supports 6 LLM providers (OpenAI, Azure OpenAI, Google Gemini, Anthropic, Ollama, LM Studio)
- Used in 2 key areas: documentation synthesis and CLI commands
- **No tests** currently exist for this module

## Key Problems Identified

### Critical Issues (High Priority)
1. **No retry logic** - Single API failure crashes entire process
2. **No client caching** - Creates new client on every call
3. **Hardcoded prompts** - Users cannot customize (Issue #277)
4. **Poor error handling** - Generic ValueError exceptions
5. **No rate limiting** - Can exhaust API quotas quickly

### Code Quality Issues (Medium Priority)
1. **Legacy Azure SDK** - Uses deprecated patterns
2. **Missing type hints** - Incomplete typing
3. **Fragile JSON parsing** - Basic regex-based extraction
4. **No logging** - No visibility into LLM interactions
5. **Duplicate code** - Separate paths for Azure vs others

### Missing Features (Lower Priority)
1. **Usage tracking** - No cost/token tracking
2. **Timeout handling** - No configurable timeouts
3. **Structured outputs** - Could use JSON mode
4. **Async support** - Synchronous only
5. **Testing infrastructure** - No mocks or tests

## Implementation Plan

### Phase 1: Critical Fixes (Immediate - Next Release)

**Estimated Time: 2-4 weeks**

1. **Add Custom Exceptions** (2 hours)
   - Create LLMError hierarchy
   - Replace generic ValueError

2. **Add Retry Logic** (4 hours)
   - Implement retry_with_backoff decorator
   - Handle rate limits specially

3. **Add Custom Prompts** (6 hours) - **Issue #277**
   - Support YAML prompt files
   - Allow environment variable overrides
   - Add CLI option

4. **Add Logging** (3 hours)
   - Log requests/responses
   - Track durations
   - Avoid sensitive data

5. **Improve JSON Parsing** (3 hours)
   - Robust extraction from various formats
   - Better error messages

**Total: ~18 hours**

### Phase 2: Code Quality (Short-term - 1-2 Releases)

**Estimated Time: 4-6 weeks**

1. **Client Caching** (4 hours)
   - Use @lru_cache
   - Hash API keys for cache

2. **Modernize Azure Client** (4 hours)
   - Use AzureOpenAI class
   - Update to modern SDK

3. **Refactor Providers** (6 hours)
   - Centralize configuration
   - Use dataclasses
   - Reduce duplication

4. **Add Type Hints** (4 hours)
   - Complete typing throughout
   - Add Protocol for mocking

5. **Add Input Validation** (3 hours)
   - Validate SQL content
   - Validate responses

**Total: ~21 hours**

### Phase 3: New Features (Long-term)

**Estimated Time: 6-8 weeks**

1. **Usage Tracking** (5 hours)
   - Track token usage
   - Estimate costs
   - Provide summaries

2. **Timeout Handling** (3 hours)
   - Configurable timeouts
   - Prevent hanging

3. **Rate Limiting** (6 hours)
   - Token bucket algorithm
   - Configurable limits

4. **JSON Mode** (4 hours)
   - Use response_format where supported
   - Better structured outputs

5. **Testing Infrastructure** (10 hours)
   - Mock client
   - Unit tests
   - Integration tests

**Total: ~28 hours**

## Priority Recommendation

### Immediate (Next Release)
✅ **Custom exceptions** - Improves debugging
✅ **Retry logic** - Prevents crashes
✅ **Custom prompts** - Addresses Issue #277
✅ **Logging** - Improves observability
✅ **JSON parsing** - More robust

### Short-term (1-2 releases)
📅 **Client caching** - Performance improvement
📅 **Azure modernization** - Future-proofing
📅 **Provider refactoring** - Reduces duplication
📅 **Type hints** - Better IDE support
📅 **Input validation** - Security improvement

### Long-term
🔜 **Usage tracking** - Cost management
🔜 **Rate limiting** - Quota management
🔜 **Testing** - Quality assurance
🔜 **Async support** - Performance

## Benefits

### Immediate Benefits
- **Reliability**: Retry logic prevents crashes
- **Customization**: Users can tailor prompts
- **Debugging**: Better error messages and logging
- **Robustness**: Improved JSON parsing

### Medium-term Benefits
- **Performance**: Client caching reduces overhead
- **Maintainability**: Refactored code is easier to maintain
- **Type safety**: Better IDE support and catch errors early
- **Security**: Input validation prevents issues

### Long-term Benefits
- **Cost control**: Usage tracking helps manage expenses
- **Scalability**: Rate limiting prevents quota issues
- **Quality**: Comprehensive tests prevent regressions
- **Performance**: Async support enables parallel processing

## Risk Assessment

### Low Risk
- Custom exceptions
- Logging
- Type hints
- Input validation

### Medium Risk
- Retry logic (may change behavior)
- Custom prompts (new feature)
- Client caching (may affect existing behavior)

### High Risk
- Provider refactoring (breaking changes possible)
- Azure modernization (may affect Azure users)
- Async support (significant architectural change)

## Recommendations

1. **Start with Phase 1** - Critical fixes provide immediate value
2. **Add tests early** - Especially for retry logic and custom prompts
3. **Communicate changes** - Document new features and deprecations
4. **Iterate** - Get feedback after each phase
5. **Monitor** - Track usage and errors after deployment

## Success Metrics

- **Reliability**: % reduction in API failures
- **Adoption**: % of users using custom prompts
- **Performance**: % reduction in API calls (via caching)
- **Cost**: % reduction in LLM costs (via usage tracking)
- **Quality**: % reduction in documentation errors

## Conclusion

The LLM module is a critical component for dbt-osmosis documentation synthesis. The proposed improvements will:

1. **Improve reliability** through retry logic and better error handling
2. **Enhance customization** through custom prompts (Issue #277)
3. **Reduce costs** through usage tracking and rate limiting
4. **Improve maintainability** through refactoring and type hints
5. **Enable testing** through mock infrastructure

The plan is prioritized to deliver immediate value while setting up for long-term success.
