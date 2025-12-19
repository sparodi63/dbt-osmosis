# dbt-osmosis LLM Module: Technical Analysis and Improvement Recommendations

## Module Overview

**File:** `src/dbt_osmosis/core/llm.py`  
**Purpose:** Synthesize missing dbt documentation using OpenAI's GPT models  
**Lines of Code:** ~330  
**Dependencies:** `openai` (optional extra)

---

## Current Architecture

### Module Structure

```
llm.py
├── Private Functions (Prompt Builders)
│   ├── _create_llm_prompt_for_model_docs_as_json()  # Bulk model + columns
│   ├── _create_llm_prompt_for_column()              # Single column doc
│   └── _create_llm_prompt_for_table()               # Table description only
│
└── Public Functions (API)
    ├── generate_model_spec_as_json()    # Returns dict with description + columns
    ├── generate_column_doc()            # Returns single column description string
    └── generate_table_doc()             # Returns table description string
```

### Function Signatures

```python
def generate_model_spec_as_json(
    sql_content: str,
    upstream_docs: list[str] | None = None,
    existing_context: str | None = None,
    model_engine: str = "gpt-4o",
    temperature: float = 0.3,
) -> dict[str, Any]

def generate_column_doc(
    column_name: str,
    existing_context: str | None = None,
    table_name: str | None = None,
    upstream_docs: list[str] | None = None,
    model_engine: str = "gpt-4o",
    temperature: float = 0.7,
) -> str

def generate_table_doc(
    sql_content: str,
    table_name: str,
    upstream_docs: list[str] | None = None,
    model_engine: str = "gpt-4o",
    temperature: float = 0.7,
) -> str
```

---

## Integration Flow

### Entry Points

The LLM module is integrated into dbt-osmosis through:

1. **CLI Commands** (`cli/main.py`)
   - `dbt-osmosis yaml refactor --synthesize`
   - `dbt-osmosis yaml document --synthesize`

2. **Transform Pipeline** (`core/osmosis.py`)
   - `synthesize_missing_documentation_with_openai` transform operation

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CLI: --synthesize flag                                │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Transform Pipeline (osmosis.py lines 269-279)                          │
│                                                                          │
│  transform = (                                                          │
│      inject_missing_columns                                             │
│      >> remove_columns_not_in_database                                  │
│      >> inherit_upstream_column_knowledge                               │
│      >> sort_columns_as_configured                                      │
│      >> synchronize_data_types                                          │
│  )                                                                       │
│  if synthesize:                                                          │
│      transform >>= synthesize_missing_documentation_with_openai   ◄─────┤
│                                                                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  synthesize_missing_documentation_with_openai (lines 2326-2438)         │
│                                                                          │
│  For each node (topologically sorted):                                   │
│  1. First run inherit_upstream_column_knowledge(context, node)          │
│     → This REDUCES LLM calls by inheriting existing docs first!         │
│                                                                          │
│  2. Count undocumented columns                                          │
│                                                                          │
│  3. Build upstream_docs context from depends_on_nodes                   │
│     → Max 20 columns per dependency                                     │
│     → Max 100 total context lines                                       │
│                                                                          │
│  4. Decision Branch:                                                     │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  IF undocumented > 10 columns:                              │     │
│     │    → Call generate_model_spec_as_json() [BULK]              │     │
│     │    → Single API call for all columns                        │     │
│     └─────────────────────────────────────────────────────────────┘     │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  ELSE (≤10 undocumented columns):                           │     │
│     │    → Call generate_table_doc() for table description        │     │
│     │    → Call generate_column_doc() for each column [N calls]   │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                                                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  llm.py Functions                                                        │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  generate_model_spec_as_json()                                   │   │
│  │  1. Build prompt with SQL + upstream docs + context              │   │
│  │  2. Call openai.chat.completions.create()                        │   │
│  │  3. Parse JSON response                                          │   │
│  │  4. Return {"description": "...", "columns": [...]}              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  generate_column_doc() / generate_table_doc()                    │   │
│  │  1. Build prompt with context                                    │   │
│  │  2. Call openai.chat.completions.create()                        │   │
│  │  3. Return plain text string                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Smart Optimization: Topological Ordering

The integration in `synthesize_missing_documentation_with_openai` is clever:

```python
# Line 2347-2349
# since we are topologically sorted, we continually pass down synthesized 
# knowledge leveraging our inheritance system
# which minimizes synthesis requests -- in some cases by an order of magnitude
_ = inherit_upstream_column_knowledge(context, node)
```

This means:
1. Nodes are processed in dependency order (sources → staging → marts)
2. Before LLM synthesis, inheritance is attempted first
3. If a parent was just synthesized, its docs propagate to children
4. Only truly novel columns need LLM synthesis

---

## Current Limitations

### 1. Hard-coded OpenAI Dependency
```python
import openai  # Line 8

response = openai.chat.completions.create(...)  # Lines 203, 245, 277
```
- No abstraction layer for different LLM providers
- Cannot use Anthropic, local models, or Azure OpenAI

### 2. No Configuration Options
- Model selection only via `model_engine` parameter (default: `gpt-4o`)
- No way to configure via `dbt_project.yml` or environment variables (except `OSMOSIS_LLM_MAX_SQL_CHARS`)
- Temperature is hard-coded per function

### 3. Limited Error Handling
```python
# Current error handling (lines 209-216)
if content is None:
    raise ValueError("OpenAI returned an empty response")
try:
    data = json.loads(content)
except json.JSONDecodeError:
    raise ValueError("OpenAI returned invalid JSON:\n" + content)
```
- No retry logic for transient failures
- No rate limiting handling
- No timeout configuration

### 4. No Cost Tracking
- No token counting or cost estimation
- No way to set budget limits

### 5. No Caching
- Same column/table documented repeatedly wastes API calls
- No persistence of generated docs between runs

### 6. Synchronous API Calls
- Each LLM call blocks the thread
- With many columns, this becomes slow
- Thread pool in osmosis.py helps but could be better

---

## Improvement Recommendations

### Priority 1: Provider Abstraction Layer

Create a pluggable LLM backend system:

```python
# llm/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any

@dataclass
class LLMConfig:
    """Configuration for LLM providers."""
    provider: str = "openai"  # openai, anthropic, ollama, azure
    model: str = "gpt-4o"
    temperature: float = 0.5
    max_tokens: int = 2000
    timeout: float = 60.0
    max_retries: int = 3
    api_key: str | None = None  # Falls back to env var
    base_url: str | None = None  # For Azure/custom endpoints

class LLMProvider(ABC):
    """Abstract base class for LLM providers."""
    
    def __init__(self, config: LLMConfig):
        self.config = config
    
    @abstractmethod
    def generate(self, messages: list[dict[str, str]]) -> str:
        """Generate completion from messages."""
        pass
    
    @abstractmethod
    def generate_json(self, messages: list[dict[str, str]]) -> dict[str, Any]:
        """Generate structured JSON output."""
        pass


# llm/providers/openai.py
class OpenAIProvider(LLMProvider):
    def __init__(self, config: LLMConfig):
        super().__init__(config)
        import openai
        self.client = openai.OpenAI(
            api_key=config.api_key,
            base_url=config.base_url,
            timeout=config.timeout,
        )
    
    def generate(self, messages: list[dict[str, str]]) -> str:
        response = self.client.chat.completions.create(
            model=self.config.model,
            messages=messages,
            temperature=self.config.temperature,
            max_tokens=self.config.max_tokens,
        )
        return response.choices[0].message.content or ""
    
    def generate_json(self, messages: list[dict[str, str]]) -> dict[str, Any]:
        # Use JSON mode if available
        response = self.client.chat.completions.create(
            model=self.config.model,
            messages=messages,
            temperature=self.config.temperature,
            response_format={"type": "json_object"},
        )
        return json.loads(response.choices[0].message.content or "{}")


# llm/providers/anthropic.py
class AnthropicProvider(LLMProvider):
    def __init__(self, config: LLMConfig):
        super().__init__(config)
        import anthropic
        self.client = anthropic.Anthropic(api_key=config.api_key)
    
    def generate(self, messages: list[dict[str, str]]) -> str:
        # Convert messages format
        system = next((m["content"] for m in messages if m["role"] == "system"), "")
        user_msgs = [m for m in messages if m["role"] != "system"]
        
        response = self.client.messages.create(
            model=self.config.model,  # claude-3-5-sonnet-20241022
            max_tokens=self.config.max_tokens,
            system=system,
            messages=user_msgs,
        )
        return response.content[0].text


# llm/providers/ollama.py
class OllamaProvider(LLMProvider):
    """Local LLM via Ollama."""
    
    def __init__(self, config: LLMConfig):
        super().__init__(config)
        self.base_url = config.base_url or "http://localhost:11434"
    
    def generate(self, messages: list[dict[str, str]]) -> str:
        import httpx
        response = httpx.post(
            f"{self.base_url}/api/chat",
            json={
                "model": self.config.model,  # llama3.2, codellama, etc.
                "messages": messages,
                "stream": False,
            },
            timeout=self.config.timeout,
        )
        return response.json()["message"]["content"]
```

### Priority 2: Configuration via dbt_project.yml

Allow configuration in the dbt project:

```yaml
# dbt_project.yml
vars:
  dbt-osmosis:
    llm:
      provider: "anthropic"  # or openai, ollama, azure
      model: "claude-3-5-sonnet-20241022"
      temperature: 0.5
      max_tokens: 2000
      # Provider-specific
      base_url: null  # For Azure/Ollama
      # Cost control
      max_cost_per_run: 5.00  # USD
      dry_run: false  # Just estimate, don't call API
```

### Priority 3: Retry Logic and Error Handling

```python
import tenacity
from tenacity import (
    retry, 
    stop_after_attempt, 
    wait_exponential,
    retry_if_exception_type
)

class LLMProvider(ABC):
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=60),
        retry=retry_if_exception_type((
            openai.RateLimitError,
            openai.APIConnectionError,
            openai.APITimeoutError,
        ))
    )
    def generate_with_retry(self, messages: list[dict[str, str]]) -> str:
        return self.generate(messages)
```

### Priority 4: Token Counting and Cost Estimation

```python
import tiktoken

class CostTracker:
    """Track and estimate LLM API costs."""
    
    PRICING = {
        "gpt-4o": {"input": 0.005, "output": 0.015},  # per 1K tokens
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "claude-3-5-sonnet-20241022": {"input": 0.003, "output": 0.015},
    }
    
    def __init__(self, model: str):
        self.model = model
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.encoder = tiktoken.encoding_for_model(model)
    
    def estimate_tokens(self, text: str) -> int:
        return len(self.encoder.encode(text))
    
    def record_usage(self, input_tokens: int, output_tokens: int):
        self.total_input_tokens += input_tokens
        self.total_output_tokens += output_tokens
    
    @property
    def total_cost(self) -> float:
        prices = self.PRICING.get(self.model, {"input": 0, "output": 0})
        return (
            (self.total_input_tokens / 1000) * prices["input"] +
            (self.total_output_tokens / 1000) * prices["output"]
        )
    
    def check_budget(self, max_cost: float) -> bool:
        return self.total_cost < max_cost
```

### Priority 5: Response Caching

```python
import hashlib
import json
from pathlib import Path

class LLMCache:
    """Cache LLM responses to avoid redundant API calls."""
    
    def __init__(self, cache_dir: Path | None = None):
        self.cache_dir = cache_dir or Path.home() / ".cache" / "dbt-osmosis" / "llm"
        self.cache_dir.mkdir(parents=True, exist_ok=True)
    
    def _hash_key(self, messages: list[dict], model: str) -> str:
        content = json.dumps({"messages": messages, "model": model}, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()[:16]
    
    def get(self, messages: list[dict], model: str) -> str | None:
        key = self._hash_key(messages, model)
        cache_file = self.cache_dir / f"{key}.json"
        if cache_file.exists():
            return json.loads(cache_file.read_text())["response"]
        return None
    
    def set(self, messages: list[dict], model: str, response: str):
        key = self._hash_key(messages, model)
        cache_file = self.cache_dir / f"{key}.json"
        cache_file.write_text(json.dumps({
            "messages": messages,
            "model": model,
            "response": response,
        }))
```

### Priority 6: Async Support for Better Throughput

```python
import asyncio
from typing import Coroutine

class AsyncLLMProvider(LLMProvider):
    """Async variant for parallel LLM calls."""
    
    async def generate_async(self, messages: list[dict[str, str]]) -> str:
        # Implementation varies by provider
        pass
    
    async def batch_generate(
        self, 
        message_batches: list[list[dict[str, str]]]
    ) -> list[str]:
        """Generate multiple completions concurrently."""
        tasks = [self.generate_async(msgs) for msgs in message_batches]
        return await asyncio.gather(*tasks)


# In synthesize function:
async def synthesize_columns_async(
    provider: AsyncLLMProvider,
    columns: list[str],
    context: str
) -> dict[str, str]:
    """Synthesize multiple column docs in parallel."""
    messages_batch = [
        _create_llm_prompt_for_column(col, context) 
        for col in columns
    ]
    results = await provider.batch_generate(messages_batch)
    return dict(zip(columns, results))
```

### Priority 7: CLI Enhancements

```python
# In cli/main.py
@click.option(
    "--llm-provider",
    type=click.Choice(["openai", "anthropic", "ollama", "azure"]),
    default="openai",
    help="LLM provider to use for documentation synthesis.",
)
@click.option(
    "--llm-model",
    type=click.STRING,
    default=None,
    help="Specific model to use (e.g., gpt-4o-mini, claude-3-5-sonnet).",
)
@click.option(
    "--llm-max-cost",
    type=click.FLOAT,
    default=None,
    help="Maximum cost in USD for LLM calls. Stops if exceeded.",
)
@click.option(
    "--llm-dry-run",
    is_flag=True,
    help="Estimate LLM costs without making actual API calls.",
)
@click.option(
    "--llm-cache/--no-llm-cache",
    default=True,
    help="Enable/disable caching of LLM responses.",
)
```

---

## Proposed New Module Structure

```
dbt_osmosis/
├── core/
│   ├── llm/
│   │   ├── __init__.py          # Public API exports
│   │   ├── base.py              # Abstract provider, config dataclass
│   │   ├── prompts.py           # Prompt templates (moved from current llm.py)
│   │   ├── cache.py             # Response caching
│   │   ├── cost.py              # Token counting, cost tracking
│   │   └── providers/
│   │       ├── __init__.py      # Provider factory
│   │       ├── openai.py        # OpenAI/Azure provider
│   │       ├── anthropic.py     # Anthropic Claude provider
│   │       └── ollama.py        # Local Ollama provider
│   └── osmosis.py               # Uses llm/ package
```

---

## Migration Path

### Phase 1: Backward-Compatible Refactor
1. Create `llm/base.py` with abstract interface
2. Move current OpenAI code to `llm/providers/openai.py`
3. Keep `llm.py` as facade for backward compatibility
4. Add deprecation warnings

### Phase 2: Add New Providers
1. Implement Anthropic provider
2. Implement Ollama provider
3. Add provider factory with config

### Phase 3: Enhanced Features
1. Add caching
2. Add cost tracking
3. Add CLI options
4. Add async support

### Phase 4: Remove Deprecated API
1. Remove old `llm.py` facade
2. Update documentation
3. Major version bump

---

## Example: Refactored Public API

```python
# New usage in osmosis.py

from dbt_osmosis.core.llm import (
    LLMConfig,
    get_provider,
    generate_model_docs,
    generate_column_doc,
)

def synthesize_missing_documentation(
    context: YamlRefactorContext, 
    node: ResultNode | None = None
) -> None:
    # Get config from dbt_project.yml or defaults
    llm_config = LLMConfig.from_dbt_vars(context.project.runtime_cfg.vars)
    
    # Get appropriate provider
    provider = get_provider(llm_config)
    
    # Check budget before proceeding
    if llm_config.max_cost and not provider.cost_tracker.check_budget(llm_config.max_cost):
        logger.warning("LLM budget exceeded, skipping synthesis")
        return
    
    # Generate with caching
    if undocumented_count > 10:
        spec = generate_model_docs(
            provider,
            sql_content=node.compiled_sql,
            upstream_docs=upstream_docs,
            use_cache=llm_config.use_cache,
        )
    else:
        for column in undocumented_columns:
            doc = generate_column_doc(
                provider,
                column_name=column.name,
                context=context_str,
                use_cache=llm_config.use_cache,
            )
```

---

## Summary

The current LLM module is functional but tightly coupled to OpenAI. The main improvements needed are:

| Priority | Improvement | Effort | Impact |
|----------|-------------|--------|--------|
| 1 | Provider abstraction | Medium | High |
| 2 | dbt_project.yml config | Low | Medium |
| 3 | Retry/error handling | Low | High |
| 4 | Cost tracking | Medium | Medium |
| 5 | Response caching | Low | High |
| 6 | Async support | High | Medium |
| 7 | CLI options | Low | Medium |

The recommended approach is to implement these incrementally, maintaining backward compatibility until a major version bump.
