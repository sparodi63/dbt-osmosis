# LLM Module Improvements Plan

## Current State Analysis

**File**: `src/dbt_osmosis/core/llm.py`
**Lines**: 489
**Purpose**: LLM-powered documentation synthesis for dbt models and columns

### Supported Providers
- OpenAI
- Azure OpenAI
- Google Gemini
- Anthropic
- Ollama (local)
- LM Studio (local)

### Current Usage
The module is used in:
1. `src/dbt_osmosis/core/transforms.py` - for synthesizing documentation
2. `src/dbt_osmosis/cli/main.py` - for CLI commands

### Key Functions
- `get_llm_client()` - Creates LLM client based on provider
- `generate_model_spec_as_json()` - Generates JSON spec for models
- `generate_column_doc()` - Generates documentation for columns
- `generate_table_doc()` - Generates documentation for tables

---

## Issues Identified

### 1. Architecture Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| No client caching | Medium | Creates new client on every API call |
| Duplicate code | Medium | Azure vs other providers have separate code paths |
| No retry logic | High | API failures cause immediate crashes |
| No rate limiting | High | Can exhaust API quotas quickly |
| Synchronous only | Medium | No async support for parallel processing |
| No dependency injection | Medium | Hard to test without mocking |

### 2. Code Quality Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| Hardcoded prompts | Medium | Users cannot customize prompts (Issue #277) |
| Legacy Azure SDK | Medium | Uses deprecated `ChatCompletion.create` pattern |
| Missing type hints | Low | Some functions lack complete typing |
| Fragile JSON parsing | Medium | Basic regex for markdown fence removal |
| No logging | Medium | LLM interactions not logged for debugging |
| Poor error messages | Medium | Lack context for debugging |
| No input validation | Low | SQL content sent directly to API |

### 3. Missing Features

| Feature | Priority | Description |
|---------|----------|-------------|
| Custom prompts | High | Allow user-defined prompt templates (Issue #277) |
| Retry with backoff | High | Handle transient API failures |
| Cost tracking | Medium | Estimate/track token usage and costs |
| Structured outputs | Medium | Use JSON mode where supported |
| Timeout handling | Medium | Prevent hanging on slow responses |
| Batch processing | Low | Process multiple items in single request |
| Streaming | Low | Stream responses for large outputs |
| Async support | Low | Enable parallel LLM calls |
| Usage statistics | Medium | Track API usage and costs |

### 4. Error Handling Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| Generic exceptions | Medium | All errors raise `ValueError` |
| Poor error messages | Medium | Lack context for debugging |
| No recovery options | Medium | Failures stop entire process |
| No rate limit handling | High | No backoff when rate limited |

### 5. Security Considerations

| Issue | Severity | Description |
|-------|----------|-------------|
| API key exposure | Medium | Keys could appear in error logs |
| No input sanitization | Low | SQL content sent directly to API |
| No validation | Low | No schema validation for responses |

---

## Improvement Plan

### Phase 1: Critical Fixes (High Priority)

#### 1.1 Add Custom Exceptions

**Problem**: All errors use generic `ValueError`, making debugging difficult.

**Solution**: Create custom exception hierarchy.

```python
class LLMError(Exception):
    """Base exception for LLM operations."""
    pass

class LLMConfigError(LLMError):
    """Raised when LLM configuration is invalid."""
    pass

class LLMAPIError(LLMError):
    """Raised when LLM API returns an error."""
    pass

class LLMRateLimitError(LLMAPIError):
    """Raised when rate limit is exceeded."""
    pass

class LLMResponseError(LLMError):
    """Raised when LLM response is invalid."""
    pass
```

#### 1.2 Add Retry Logic with Exponential Backoff

**Problem**: Single API failure crashes the entire documentation process.

**Solution**: Implement retry decorator with exponential backoff.

```python
import time
from functools import wraps

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
):
    """Decorator to retry LLM calls with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except LLMRateLimitError as e:
                    # Special handling for rate limits
                    delay = min(base_delay * (exponential_base ** attempt), max_delay)
                    logger.warning(
                        f"Rate limit hit, retrying in {delay:.1f}s (attempt {attempt + 1}/{max_retries})"
                    )
                    time.sleep(delay)
                    last_exception = e
                except LLMAPIError as e:
                    if attempt < max_retries - 1:
                        delay = base_delay * (exponential_base ** attempt)
                        logger.warning(
                            f"LLM error: {type(e).__name__}, retrying in {delay:.1f}s"
                        )
                        time.sleep(delay)
                        last_exception = e
                    else:
                        raise
                except Exception as e:
                    # Non-retryable errors
                    raise LLMError(f"Unexpected error in LLM call: {e}") from e
            
            # If we exhausted retries
            raise last_exception or LLMError("LLM call failed after retries")
        return wrapper
    return decorator
```

#### 1.3 Cache LLM Client

**Problem**: New client created on every call, wasting resources.

**Solution**: Implement singleton pattern with lazy initialization.

```python
from functools import lru_cache
from dataclasses import dataclass
import hashlib

@dataclass(frozen=True)
class LLMClientConfig:
    """Immutable configuration for LLM client caching."""
    provider: str
    api_key_hash: str  # Hash of API key for cache key
    base_url: str | None
    model: str

@lru_cache(maxsize=4)
def _get_cached_client(config: LLMClientConfig) -> tuple[Any, str]:
    """Get or create cached LLM client."""
    provider = config.provider
    
    if provider == "openai":
        client = OpenAI(api_key=_get_api_key_for_provider(provider))
        return client, config.model
    
    elif provider == "azure-openai":
        # Azure client creation
        return _create_azure_client()
    
    # Other providers...

def get_llm_client() -> tuple[Any, str]:
    """Get LLM client, using cache when possible."""
    provider = os.getenv("LLM_PROVIDER", "openai").lower()
    api_key = _get_api_key_for_provider(provider)
    
    config = LLMClientConfig(
        provider=provider,
        api_key_hash=hashlib.sha256(api_key.encode()).hexdigest()[:16],
        base_url=_get_base_url_for_provider(provider),
        model=_get_model_for_provider(provider),
    )
    
    return _get_cached_client(config)
```

#### 1.4 Add Custom Prompt Support (Issue #277)

**Problem**: Users cannot customize prompts for company-specific needs.

**Solution**: Support external prompt templates.

```python
from pathlib import Path
from string import Template
import yaml

# Default prompts (current behavior)
DEFAULT_PROMPTS = {
    "model_system": """You are a helpful SQL Developer and an Expert in dbt...""",
    "model_user": """The SQL for the model is: ...""",
    "column_system": """You are a helpful SQL Developer...""",
    "column_user": """The column name is: {column_name}...""",
    "table_system": """You are a helpful SQL Developer...""",
    "table_user": """The SQL for the model is: ...""",
}

def load_custom_prompts() -> dict[str, str]:
    """Load custom prompts from file or environment."""
    prompts = DEFAULT_PROMPTS.copy()

    # Check for prompt file
    prompt_file = os.getenv("OSMOSIS_LLM_PROMPT_FILE")
    if prompt_file and Path(prompt_file).exists():
        with open(prompt_file) as f:
            custom = yaml.safe_load(f)
            prompts.update(custom)

    # Check for individual prompt overrides
    for key in DEFAULT_PROMPTS:
        env_key = f"OSMOSIS_LLM_PROMPT_{key.upper()}"
        if env_value := os.getenv(env_key):
            prompts[key] = env_value

    return prompts

def _create_llm_prompt_for_column(
    column_name: str,
    existing_context: str | None = None,
    table_name: str | None = None,
    upstream_docs: list[str] | None = None,
    custom_prompts: dict[str, str] | None = None,
) -> list[dict[str, str]]:
    """Build prompts with custom template support."""
    prompts = custom_prompts or load_custom_prompts()

    system_template = Template(prompts["column_system"])
    user_template = Template(prompts["column_user"])

    system_prompt = system_template.safe_substitute(
        table_name=table_name or "unknown",
    )
    user_prompt = user_template.safe_substitute(
        column_name=column_name,
        existing_context=existing_context or "(none)",
        upstream_docs=os.linesep.join(upstream_docs or []),
    )

    return [
        {"role": "system", "content": system_prompt.strip()},
        {"role": "user", "content": user_prompt.strip()},
    ]
```

**Prompt File Format** (`prompts.yaml`):
```yaml
# Custom prompts for dbt-osmosis LLM documentation

model_system: |
  You are a data documentation specialist at {company_name}.
  Follow our documentation standards:
  - Use business terminology
  - Reference data lineage
  - Include data quality notes

column_system: |
  Document this column following {company_name} standards.
  Be concise. Use active voice.
```

---

### Phase 2: Code Quality Improvements (Medium Priority)

#### 2.1 Modernize Azure OpenAI Client

**Problem**: Uses deprecated legacy SDK pattern.

**Current Code**:
```python
# Legacy pattern
openai.api_type = "azure-openai"
openai.api_base = os.getenv("AZURE_OPENAI_BASE_URL")
response = client.ChatCompletion.create(engine=model_engine, ...)
```

**Proposed Code**:
```python
from openai import AzureOpenAI

def _create_azure_client() -> tuple[AzureOpenAI, str]:
    """Create Azure OpenAI client using modern SDK."""
    client = AzureOpenAI(
        api_key=os.getenv("AZURE_OPENAI_API_KEY"),
        api_version=os.getenv("AZURE_OPENAI_API_VERSION", "2024-02-15-preview"),
        azure_endpoint=os.getenv("AZURE_OPENAI_BASE_URL"),
    )
    model = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME")
    return client, model
```

#### 2.2 Add Logging

**Problem**: No visibility into LLM interactions.

**Solution**: Add structured logging.

```python
import dbt_osmosis.core.logger as logger

def _log_llm_request(
    provider: str,
    model: str,
    prompt_tokens: int | None = None,
    operation: str = "unknown",
):
    """Log LLM request details (without sensitive data)."""
    logger.debug(
        ":robot: LLM Request: provider=%s model=%s operation=%s tokens=%s",
        provider, model, operation, prompt_tokens or "unknown"
    )

def _log_llm_response(
    provider: str,
    model: str,
    response_tokens: int | None = None,
    duration_ms: float | None = None,
):
    """Log LLM response details."""
    logger.debug(
        ":white_check_mark: LLM Response: provider=%s model=%s tokens=%s duration=%sms",
        provider, model, response_tokens or "unknown", duration_ms or "unknown"
    )
```

#### 2.3 Improve JSON Parsing

**Problem**: Fragile regex-based markdown fence removal.

**Solution**: More robust JSON extraction.

```python
import re

def _extract_json_from_response(content: str) -> dict[str, Any]:
    """
    Extract JSON from LLM response, handling various formats.

    Handles:
    - Plain JSON
    - JSON wrapped in ```json ... ```
    - JSON wrapped in ``` ... ```
    - JSON with leading/trailing text
    """
    content = content.strip()

    # Try direct parsing first
    try:
        return json.loads(content)
    except json.JSONDecodeError:
        pass

    # Remove markdown code fences
    patterns = [
        r"```json\s*([\s\S]*?)\s*```",  # ```json ... ```
        r"```\s*([\s\S]*?)\s*```",       # ``` ... ```
        r"(\{[\s\S]*\})",                 # Find JSON object
    ]

    for pattern in patterns:
        match = re.search(pattern, content)
        if match:
            try:
                return json.loads(match.group(1))
            except json.JSONDecodeError:
                continue

    # Last resort: find first { and last }
    start = content.find("{")
    end = content.rfind("}")
    if start != -1 and end != -1 and end > start:
        try:
            return json.loads(content[start:end + 1])
        except json.JSONDecodeError:
            pass

    raise LLMResponseError(
        f"Could not extract valid JSON from LLM response:\n{content[:500]}..."
    )
```

#### 2.4 Refactor Provider Configuration

**Problem**: Duplicate configuration logic scattered throughout.

**Solution**: Centralize provider configuration.

```python
from dataclasses import dataclass

@dataclass
class ProviderConfig:
    """Configuration for an LLM provider."""
    name: str
    required_env_vars: list[str]
    optional_env_vars: dict[str, str]  # var -> default
    client_class: type
    uses_engine: bool = False  # Azure uses 'engine' instead of 'model'

PROVIDERS: dict[str, ProviderConfig] = {
    "openai": ProviderConfig(
        name="OpenAI",
        required_env_vars=["OPENAI_API_KEY"],
        optional_env_vars={"OPENAI_MODEL": "gpt-4o"},
        client_class=OpenAI,
    ),
    "azure-openai": ProviderConfig(
        name="Azure OpenAI",
        required_env_vars=[
            "AZURE_OPENAI_API_KEY",
            "AZURE_OPENAI_BASE_URL",
            "AZURE_OPENAI_DEPLOYMENT_NAME",
        ],
        optional_env_vars={"AZURE_OPENAI_API_VERSION": "2024-02-15-preview"},
        client_class=AzureOpenAI,
        uses_engine=True,
    ),
    # ... other providers
}

def get_llm_client() -> tuple[Any, str]:
    """Get LLM client based on provider configuration."""
    provider_name = os.getenv("LLM_PROVIDER", "openai").lower()

    if provider_name not in PROVIDERS:
        valid = ", ".join(PROVIDERS.keys())
        raise LLMConfigError(f"Invalid provider '{provider_name}'. Valid: {valid}")

    config = PROVIDERS[provider_name]

    # Validate required vars
    missing = [v for v in config.required_env_vars if not os.getenv(v)]
    if missing:
        raise LLMConfigError(
            f"Missing environment variables for {config.name}: {', '.join(missing)}"
        )

    # Build client kwargs
    # ... provider-specific logic
```

#### 2.5 Add Type Hints

**Problem**: Some functions lack complete type hints.

**Solution**: Add comprehensive type hints throughout.

```python
from typing import Any, Optional, Protocol

class LLMClient(Protocol):
    """Protocol for LLM clients (enables mocking)."""

    def chat_completions_create(
        self,
        model: str,
        messages: list[dict[str, str]],
        temperature: float = 0.7,
        **kwargs: Any,
    ) -> Any:
        """Generate chat completion."""
        ...

def generate_model_spec_as_json(
    sql_content: str,
    upstream_docs: Optional[list[str]] = None,
    existing_context: Optional[str] = None,
    temperature: float = 0.3,
) -> dict[str, Any]:
    """Generate model spec with full type hints."""
    ...
```

---

### Phase 3: New Features (Lower Priority)

#### 3.1 Cost Tracking

**Solution**: Track token usage and estimate costs.

```python
from dataclasses import dataclass, field

@dataclass
class LLMUsageStats:
    """Track LLM usage statistics."""
    total_requests: int = 0
    total_prompt_tokens: int = 0
    total_completion_tokens: int = 0
    total_cost_usd: float = 0.0
    errors: int = 0

    # Approximate costs per 1K tokens (update as needed)
    COSTS = {
        "gpt-4o": {"input": 0.005, "output": 0.015},
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
    }

    def record_usage(self, model: str, prompt_tokens: int, completion_tokens: int):
        """Record token usage and estimate cost."""
        self.total_requests += 1
        self.total_prompt_tokens += prompt_tokens
        self.total_completion_tokens += completion_tokens

        if model in self.COSTS:
            costs = self.COSTS[model]
            self.total_cost_usd += (
                (prompt_tokens / 1000) * costs["input"] +
                (completion_tokens / 1000) * costs["output"]
            )

    def summary(self) -> str:
        """Return usage summary."""
        return (
            f"LLM Usage: {self.total_requests} requests, "
            f"{self.total_prompt_tokens + self.total_completion_tokens} tokens, "
            f"~${self.total_cost_usd:.4f} USD"
        )

# Global stats (or attach to context)
_usage_stats = LLMUsageStats()

def get_usage_stats() -> LLMUsageStats:
    """Get current usage statistics."""
    return _usage_stats
```

#### 3.2 Structured Output Mode (JSON Mode)

**Solution**: Use native JSON mode for compatible models.

```python
def generate_model_spec_as_json(
    sql_content: str,
    upstream_docs: list[str] | None = None,
    existing_context: str | None = None,
    temperature: float = 0.3,
    use_json_mode: bool = True,
) -> dict[str, Any]:
    """Generate model spec using structured output when available."""
    client, model_engine = get_llm_client()
    messages = _create_llm_prompt_for_model_docs_as_json(...)

    # Models that support JSON mode
    json_mode_models = {"gpt-4o", "gpt-4o-mini", "gpt-4-turbo", "gpt-3.5-turbo"}

    kwargs = {
        "model": model_engine,
        "messages": messages,
        "temperature": temperature,
    }

    # Use JSON mode if supported
    if use_json_mode and any(m in model_engine for m in json_mode_models):
        kwargs["response_format"] = {"type": "json_object"}

    response = client.chat.completions.create(**kwargs)
    # ... rest of function
```

#### 3.3 Timeout Handling

**Solution**: Add configurable timeouts.

```python
import httpx

def get_llm_client() -> tuple[Any, str]:
    """Get LLM client with timeout configuration."""
    timeout = float(os.getenv("OSMOSIS_LLM_TIMEOUT", "120"))

    client = OpenAI(
        api_key=api_key,
        timeout=httpx.Timeout(timeout, connect=10.0),
    )
    # ...
```

#### 3.4 Rate Limiting

**Solution**: Implement token bucket rate limiter.

```python
import threading
import time

class RateLimiter:
    """Token bucket rate limiter for API calls."""

    def __init__(self, requests_per_minute: int = 60):
        self.rate = requests_per_minute
        self.tokens = requests_per_minute
        self.last_update = time.time()
        self.lock = threading.Lock()

    def acquire(self):
        """Block until a request can be made."""
        with self.lock:
            now = time.time()
            # Refill tokens
            elapsed = now - self.last_update
            self.tokens = min(self.rate, self.tokens + elapsed * (self.rate / 60))
            self.last_update = now

            if self.tokens < 1:
                sleep_time = (1 - self.tokens) * (60 / self.rate)
                time.sleep(sleep_time)
                self.tokens = 0
            else:
                self.tokens -= 1

# Global rate limiter
_rate_limiter = RateLimiter(
    requests_per_minute=int(os.getenv("OSMOSIS_LLM_RATE_LIMIT", "60"))
)
```

---

### Phase 4: Testing Improvements

#### 4.1 Add Mock Support

**Solution**: Enable dependency injection for testing.

```python
class MockLLMClient:
    """Mock client for testing."""

    def __init__(self, responses: dict[str, str] | None = None):
        self.responses = responses or {}
        self.calls: list[dict] = []

    def chat_completions_create(self, model: str, messages: list[dict], **kwargs) -> Any:
        self.calls.append({"messages": messages, "model": model})
        # Return mock response based on content
        user_msg = messages[-1]["content"]
        
        # Mock response
        class MockResponse:
            def __init__(self, content):
                self.choices = [
                    type('obj', (object,), {
                        'message': type('obj', (object,), {'content': content})()
                    })()
                ]
        
        return MockResponse(self.responses.get(user_msg, '{"description": "Mock", "columns": []}'))
```

#### 4.2 Add Unit Tests

**Test Structure**:
```python
# tests/core/test_llm.py

import pytest
from dbt_osmosis.core.llm import (
    generate_model_spec_as_json,
    generate_column_doc,
    generate_table_doc,
)
from dbt_osmosis.core.llm.exceptions import LLMConfigError

def test_custom_prompts_loading():
    """Test loading custom prompts from file."""
    # Create temp prompt file
    import tempfile
    import os
    
    with tempfile.NamedTemporaryFile(mode='w', suffix='.yaml', delete=False) as f:
        f.write("column_system: 'Custom prompt'")
        temp_path = f.name
    
    try:
        os.environ["OSMOSIS_LLM_PROMPT_FILE"] = temp_path
        # Test that custom prompt is loaded
        # ...
    finally:
        del os.environ["OSMOSIS_LLM_PROMPT_FILE"]
        os.unlink(temp_path)

def test_retry_logic():
    """Test retry with backoff decorator."""
    from dbt_osmosis.core.llm.retry import retry_with_backoff
    
    call_count = [0]
    
    @retry_with_backoff(max_retries=3, base_delay=0.1)
    def failing_function():
        call_count[0] += 1
        if call_count[0] < 3:
            raise LLMAPIError("Temporary error")
        return "success"
    
    result = failing_function()
    assert result == "success"
    assert call_count[0] == 3

def test_json_extraction():
    """Test JSON extraction from various formats."""
    from dbt_osmosis.core.llm import _extract_json_from_response
    
    # Test plain JSON
    assert _extract_json_from_response('{"key": "value"}') == {"key": "value"}
    
    # Test with markdown fences
    result = _extract_json_from_response('```json\n{"key": "value"}\n```')
    assert result == {"key": "value"}
    
    # Test with extra text
    result = _extract_json_from_response('Some text {"key": "value"} more text')
    assert result == {"key": "value"}

def test_missing_env_vars():
    """Test error handling for missing environment variables."""
    # Save original env
    original = os.environ.copy()
    
    try:
        # Clear all LLM env vars
        for key in list(os.environ.keys()):
            if key.startswith("OPENAI") or key.startswith("AZURE") or key.startswith("LLM"):
                del os.environ[key]
        
        with pytest.raises(LLMConfigError):
            get_llm_client()
    finally:
        # Restore original env
        os.environ.clear()
        os.environ.update(original)
```

---

## Implementation Checklist

### Phase 1: Critical Fixes
- [ ] Add custom exception classes
- [ ] Implement retry_with_backoff decorator
- [ ] Add client caching with @lru_cache
- [ ] Implement custom prompt loading from file/env
- [ ] Add --llm-prompt-file CLI option
- [ ] Update all functions to use new exceptions

### Phase 2: Code Quality
- [ ] Modernize Azure OpenAI client
- [ ] Add logging for LLM requests/responses
- [ ] Improve JSON extraction robustness
- [ ] Refactor provider configuration into dataclasses
- [ ] Add type hints throughout
- [ ] Add input validation

### Phase 3: New Features
- [ ] Implement usage tracking
- [ ] Add JSON mode support
- [ ] Add timeout configuration
- [ ] Implement rate limiting
- [ ] Add --llm-stats flag to show usage
- [ ] Add --llm-timeout CLI option

### Phase 4: Testing
- [ ] Add LLMClient protocol
- [ ] Create MockLLMClient
- [ ] Add unit tests for all functions
- [ ] Add integration tests with mock
- [ ] Test retry logic
- [ ] Test rate limiting
- [ ] Test custom prompts

---

## New Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OSMOSIS_LLM_PROMPT_FILE` | None | Path to custom prompts YAML |
| `OSMOSIS_LLM_TIMEOUT` | 120 | Request timeout in seconds |
| `OSMOSIS_LLM_RATE_LIMIT` | 60 | Max requests per minute |
| `OSMOSIS_LLM_MAX_RETRIES` | 3 | Number of retry attempts |
| `OSMOSIS_LLM_RETRY_DELAY` | 1.0 | Base delay between retries |
| `OSMOSIS_LLM_TRACK_USAGE` | false | Enable usage tracking |
| `OSMOSIS_LLM_USE_JSON_MODE` | true | Use JSON mode when available |

---

## New CLI Options

```bash
# Use custom prompts
dbt-osmosis yaml document --synthesize --llm-prompt-file prompts.yaml

# Show usage statistics after run
dbt-osmosis yaml document --synthesize --llm-stats

# Set timeout
dbt-osmosis yaml document --synthesize --llm-timeout 60

# Dry run (show what would be generated without calling API)
dbt-osmosis yaml document --synthesize --llm-dry-run

# Disable JSON mode
dbt-osmosis yaml document --synthesize --no-llm-json-mode
```

---

## Migration Guide

### For Existing Users

1. **No breaking changes** - existing env vars still work
2. **Azure users**: Consider updating to new SDK pattern (optional)
3. **Custom prompts**: Create `prompts.yaml` for customization
4. **Retry behavior**: New retry logic is opt-in via environment variables

### Deprecation Timeline

| Item | Deprecated | Removed |
|------|------------|---------|
| Legacy Azure SDK pattern | v1.2.0 | v2.0.0 |
| Direct `openai` module usage | v1.2.0 | v2.0.0 |

### Backwards Compatibility

All improvements are designed to be backwards compatible:
- Default behavior unchanged
- New features opt-in via env vars or CLI flags
- Existing code continues to work

---

## File Structure After Refactoring

```
src/dbt_osmosis/core/
├── llm/
│   ├── __init__.py        # Public API exports
│   ├── client.py          # LLM client abstraction
│   ├── providers.py       # Provider configurations
│   ├── prompts.py         # Prompt templates and loading
│   ├── retry.py           # Retry logic
│   ├── rate_limit.py      # Rate limiting
│   ├── usage.py           # Usage tracking
│   ├── exceptions.py      # Custom exceptions
│   └── parsing.py         # JSON parsing utilities
└── llm.py                 # Backwards-compatible facade (deprecated)
```

**Note**: The module can be refactored incrementally. The old `llm.py` can remain as a facade that imports from the new submodule structure, ensuring backwards compatibility.

---

## Summary

### Quick Wins (1-2 hours each)
1. Add custom exceptions
2. Add retry with backoff
3. Add logging
4. Improve JSON parsing
5. Add custom prompt support (Issue #277)

### Medium Effort (4-8 hours each)
1. Client caching
2. Provider refactoring
3. Type hints throughout
4. Usage tracking

### Larger Effort (1-2 days each)
1. Full module refactoring into submodules
2. Comprehensive testing suite
3. Async support
4. Rate limiting implementation

### Priority Recommendation

**Immediate (Next Release)**:
- Custom exceptions
- Retry logic
- Custom prompts (Issue #277)
- Logging

**Short-term (1-2 releases)**:
- Client caching
- JSON parsing improvements
- Provider refactoring
- Usage tracking

**Long-term**:
- Full module refactoring
- Async support
- Rate limiting
- Comprehensive tests

---

*Improvement plan for dbt-osmosis LLM module*
*Updated: December 2024*
