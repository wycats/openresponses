# Cache Control Extension for OpenResponses

**Status**: Research / Proposal  
**Date**: 2026-02-04  
**Branch**: `feat/cache-control-extension`

## Executive Summary

This document analyzes prompt caching semantics across major AI providers to inform a cache control extension for the OpenResponses specification. The goal is to standardize caching hints that are portable across providers while remaining faithful to each provider's underlying semantics.

## Problem Statement

OpenResponses aims to be a provider-agnostic API specification. However, prompt caching—a significant cost and latency optimization—is implemented differently across providers. Without a standardized approach, clients must either:

1. Forego caching benefits entirely
2. Use `providerOptions` escape hatches, losing portability
3. Implement provider-specific code paths

## Provider Landscape

### Anthropic (Claude)

**Model**: Breakpoint-based inline caching

**Semantics**:
- Cache prefixes are created in order: `tools` → `system` → `messages`
- A `cache_control` marker means "cache everything up to and including this block"
- Maximum 4 breakpoints per request
- Minimum cacheable size: 1024 tokens (Claude 3.5 Sonnet), 2048 tokens (Claude 3.5 Haiku)

**Granularity**:
| Content Type | Caching Unit |
|--------------|--------------|
| Tools | Entire `tools` array as one block |
| System | Entire system message as one block |
| Messages | Per content block within messages |

**Control Mechanism**:
```json
{
  "cache_control": {
    "type": "ephemeral"
  }
}
```

**TTL Options**:
- Default: 5 minutes
- Extended: 1 hour (via `ttl` field, currently undocumented in public API)

**Usage Reporting**:
```json
{
  "usage": {
    "input_tokens": 2264,
    "output_tokens": 503,
    "cache_creation_input_tokens": 2000,
    "cache_read_input_tokens": 264
  }
}
```

**Key Insight**: Tools and system are cached as atomic blocks. You cannot cache individual tools—marking any tool caches the entire tools array.

---

### Google Gemini

**Model**: Explicit cache objects (external to request)

**Semantics**:
- Caches are created as separate API resources via `caches.create()`
- Each cache has a name, TTL, and content
- Requests reference caches by ID via `cached_content` field
- Cache acts as an implicit prefix to the request

**Granularity**:
- Whole cache object (can contain system instructions, content, files)
- No inline breakpoint markers

**Control Mechanism**:
```python
# Create cache (separate API call)
cache = caching.CachedContent.create(
    model="gemini-1.5-flash-001",
    display_name="example_cache",
    system_instruction="You are an expert...",
    contents=[...],
    ttl=datetime.timedelta(minutes=60),
)

# Use cache in request
response = model.generate_content(
    contents=["What is..."],
    cached_content=cache.name,
)
```

**TTL Options**:
- Configurable per cache object
- Default: 1 hour
- Can be updated via `cache.update(ttl=...)`

**Usage Reporting**:
```json
{
  "usage_metadata": {
    "prompt_token_count": 1000,
    "cached_content_token_count": 900,
    "candidates_token_count": 50
  }
}
```

**Key Insight**: Gemini's model is fundamentally different—caches are first-class resources with lifecycle management, not inline hints.

**Implicit Caching**: Gemini also has automatic implicit caching (no control needed). Context is cached automatically when repeated across requests. This is read-only—no hints, just observe usage.

---

### Azure OpenAI

**Model**: Implicit/automatic caching

**Semantics**:
- No explicit cache control documented
- Automatic caching of prompt tokens
- Observable only via usage reporting

**Control Mechanism**: None documented

**Usage Reporting** (preview):
```json
{
  "usage": {
    "prompt_tokens": 1000,
    "prompt_tokens_details": {
      "cached_tokens": 800
    }
  }
}
```

**Key Insight**: Azure follows OpenAI's implicit caching model. Clients cannot influence caching; they can only observe hits.

---

### Mistral

**Model**: Not documented

No caching mechanism found in public API documentation as of 2026-02-04.

---

### Cohere

**Model**: Not documented

No caching mechanism found in public API documentation as of 2026-02-04.

---

### Amazon Bedrock

**Model**: Unknown (documentation inaccessible)

AWS Bedrock wraps multiple providers (including Anthropic). Caching behavior likely depends on underlying model, but Bedrock-specific documentation was not accessible.

---

## Analysis

### The Two Fundamental Models

| Aspect | Breakpoint Model | Object Model |
|--------|------------------|--------------|
| **Providers** | Anthropic | Gemini |
| **Cache Creation** | Inline in request | Separate API call |
| **Cache Reference** | Implicit (same request) | Explicit ID |
| **Lifecycle** | Request-scoped hints | Managed resource |
| **Granularity** | Block-level markers | Whole cache |
| **TTL Control** | Per-breakpoint | Per-object |

### The Implicit Model

Azure/OpenAI use automatic caching with no client control. This is a third category:

| Aspect | Implicit Model |
|--------|----------------|
| **Providers** | Azure OpenAI, OpenAI |
| **Cache Creation** | Automatic |
| **Cache Reference** | None |
| **Lifecycle** | Provider-managed |
| **Granularity** | Opaque |
| **TTL Control** | None |

### Common Ground

Despite different models, all providers share:

1. **Usage reporting** of cached tokens (read)
2. **Usage reporting** of cache creation tokens (write) where applicable
3. **Concept of cacheable content** (tools, system, messages)

### Irreconcilable Differences

1. **Object vs Inline**: Gemini requires external cache management; Anthropic uses inline hints
2. **Granularity**: Anthropic allows per-block caching; Gemini caches whole objects
3. **TTL Control**: Varies from none (Azure) to fine-grained (Gemini)

---

## Design Options

### Option A: Breakpoint-First

Standardize Anthropic-style inline `cache_control` markers.

**Request-level fields** (for atomic blocks):
```json
{
  "tools_cache_control": { "type": "ephemeral" },
  "instructions_cache_control": { "type": "ephemeral" }
}
```

**Content-level fields** (for message parts):
```json
{
  "type": "input_text",
  "text": "...",
  "cache_control": { "type": "ephemeral" }
}
```

**Pros**:
- Matches dominant caching-aware provider (Anthropic)
- Simple for spec—complexity in gateways
- Works for 80%+ of caching use cases

**Cons**:
- Lossy translation to Gemini (breakpoints → cache objects)
- Cannot express Gemini's cache-as-resource model
- Gemini gateways must manage cache lifecycle implicitly

**Gateway Behavior**:
| Provider | Translation |
|----------|-------------|
| Anthropic | Direct mapping |
| Gemini | Create ephemeral cache from marked content, reference in request |
| Azure | Ignore hints, report usage |
| Others | Ignore hints |

---

### Option B: Dual-Model

Support both breakpoint hints AND cache references.

**Breakpoint hints** (same as Option A):
```json
{
  "cache_control": { "type": "ephemeral" }
}
```

**Cache reference** (for Gemini-style):
```json
{
  "cached_content": "caches/abc123"
}
```

**Pros**:
- Full expressiveness for both models
- Native support for Gemini's cache-as-resource

**Cons**:
- Increased spec complexity
- Clients must understand two models
- Cache lifecycle management out of scope for request API

---

### Option C: Usage-Only (Minimal)

Standardize only the response-side: cached token reporting.

**No request-side changes**. Caching hints remain in `providerOptions`.

**Response standardization**:
```json
{
  "usage": {
    "input_tokens": 1000,
    "input_tokens_details": {
      "cached_tokens": 800,
      "cache_creation_tokens": 0
    }
  }
}
```

**Pros**:
- Minimal spec surface
- No translation complexity
- Works for all providers (implicit or explicit)

**Cons**:
- No portable caching hints
- Clients cannot optimize for caching portably
- Punts the hard problem

---

## Recommendation

**Option A (Breakpoint-First)** with the following refinements:

### 1. Request-Level Cache Control

For content cached as atomic blocks:

```typescript
interface CreateResponseRequest {
  // Existing fields...
  
  /** Cache control hint for the tools array (cached as one block) */
  tools_cache_control?: CacheControl;
  
  /** Cache control hint for instructions (cached as one block) */
  instructions_cache_control?: CacheControl;
}

interface CacheControl {
  /** Cache type. Currently only "ephemeral" is defined. */
  type: "ephemeral";
  
  /** Optional TTL hint. Provider may ignore or adjust. */
  ttl?: "5m" | "1h";
}
```

### 2. Content-Level Cache Control

For fine-grained caching within messages:

```typescript
interface InputTextContentParam {
  type: "input_text";
  text: string;
  
  /** Cache control hint for this content block */
  cache_control?: CacheControl;
}

// Similarly for InputImageContent, InputFileContent, etc.
```

### 3. Usage Reporting

Extend existing `InputTokensDetails`:

```typescript
interface InputTokensDetails {
  /** Tokens read from cache (already exists in spec as cached_tokens) */
  cached_tokens?: number;
  
  /** Tokens written to cache (new) */
  cache_creation_tokens?: number;
}
```

### 4. Gateway Translation Rules

| Provider | `tools_cache_control` | `instructions_cache_control` | Content `cache_control` |
|----------|----------------------|------------------------------|------------------------|
| Anthropic | Map to last tool's `cache_control` | Map to system message `cache_control` | Direct mapping |
| Gemini | Create tool cache object, reference in request | Create instruction cache object | Create content cache object |
| Azure | Ignore | Ignore | Ignore |
| Others | Ignore | Ignore | Ignore |

### 5. Reserved for Future

```typescript
interface CreateResponseRequest {
  /** Reserved for Gemini-style cache references (future extension) */
  cached_content?: string;
}
```

---

## Open Questions

1. **TTL Normalization**: Should OpenResponses define a standard TTL vocabulary, or allow arbitrary durations?

2. **Cache Limits**: Should the spec document provider-specific limits (max breakpoints, min tokens)?

3. **Cache Invalidation**: Gemini supports explicit cache deletion. Should OpenResponses expose this?

4. **Implicit Caching Opt-Out**: Some providers may want to disable automatic caching. Should this be exposed?

---

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [Azure OpenAI Reference (Preview)](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/reference-preview)
- [OpenAI Responses API](https://platform.openai.com/docs/api-reference/responses)
