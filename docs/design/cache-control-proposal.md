# Cache Hints for OpenResponses

**Status**: Proposal  
**Date**: 2026-02-04  
**Authors**: Yehuda Katz  
**Branch**: `feat/cache-control-extension`

## Abstract

This proposal adds portable cache hints to the OpenResponses specification. Cache hints allow clients to indicate that certain content is stable and suitable for caching, enabling cost and latency optimizations across providers.

The design prioritizes:
1. **Observable guarantees** over implementation details
2. **Semantic intent** over literal durations
3. **Graceful degradation** for providers with limited caching support

## Motivation

Prompt caching provides significant cost savings (up to 90% on cached tokens) and latency improvements. However, caching mechanisms differ substantially across providers:

- **Anthropic**: Inline breakpoint markers with prefix-based caching
- **Gemini**: Named cache objects with explicit lifecycle management
- **OpenAI/Azure**: Implicit automatic caching with no client control

Without a portable abstraction, clients must either forego caching benefits or implement provider-specific code paths.

## Design Principles

### 1. Hints, Not Commands

Cache fields express *intent*, not *mechanism*. Providers MAY honor hints according to their caching model. Providers that do not support explicit caching SHOULD ignore hints gracefully.

### 2. Observable Behavior Contracts

The specification defines guarantees in terms of observable outcomes (cache hits reported in usage), not implementation mechanisms (prefix caching vs object caching).

### 3. Semantic Categories with Normative Bounds

Duration hints use semantic categories (`ephemeral`, `session`) with specified minimum durations, following the CLDR/Intl pattern of abstracting locale-specific implementations behind normative behavior bounds.

---

## Specification

### 1. CacheHint Type

```typescript
interface CacheHint {
  /**
   * Cache hint type.
   * 
   * - "ephemeral": Content is stable for the current interaction.
   *   Suitable for caching within a single user session or request burst.
   */
  type: "ephemeral";
  
  /**
   * Duration category for cached content.
   * 
   * | Category    | Minimum | Description                              |
   * |-------------|---------|------------------------------------------|
   * | "ephemeral" | 1 min   | Very short-lived, single interaction     |
   * | "session"   | 30 min  | Session-scoped, multi-turn conversation  |
   * | "extended"  | 4 hours | Long-lived, cross-session reuse          |
   * 
   * Default: "ephemeral"
   * 
   * NORMATIVE: Providers that honor this hint MUST cache for at least
   * the specified minimum duration. Providers MAY cache longer.
   * Providers that cannot meet the minimum SHOULD behave as if no
   * duration hint was provided.
   */
  ttl?: "ephemeral" | "session" | "extended";
}
```

### 2. Request-Level Cache Hints

```typescript
interface CreateResponseRequest {
  // ... existing fields ...
  
  /**
   * Cache hints for request components.
   * 
   * These hints indicate that the specified components are stable
   * and suitable for caching. Providers MAY honor these hints to
   * optimize repeated requests with similar content.
   */
  cache?: {
    /**
     * Cache hint for the tools array.
     * 
     * When specified, indicates that the entire tools configuration
     * is stable and SHOULD be cached as a unit.
     */
    tools?: CacheHint;
    
    /**
     * Cache hint for the instructions (system message).
     * 
     * When specified, indicates that the instructions are stable
     * and SHOULD be cached.
     */
    instructions?: CacheHint;
  };
  
  /**
   * Reference to a named cache object.
   * 
   * When specified, the referenced cache content acts as an implicit
   * prefix to the request. The `input` field contains content that
   * follows the cached prefix.
   * 
   * INTERACTION WITH `cache` HINTS:
   * - `cached_content` provides the prefix (already cached)
   * - `cache` hints apply to content in `input` (new caching)
   * 
   * NORMATIVE: If specified, the cache MUST exist and be accessible.
   * Providers SHOULD return an error if the cache is not found.
   * 
   * RESERVED: This field is reserved for providers that support
   * named cache objects. Providers that only support content-addressed
   * caching SHOULD return an error or ignore this field.
   * 
   * See "Gap Analysis" section for limitations.
   */
  cached_content?: string;
}
```

### 3. Content-Level Cache Hints

```typescript
interface InputTextContentParam {
  type: "input_text";
  text: string;
  
  /**
   * Cache hint for this content block.
   * 
   * SEMANTICS:
   * Indicates that content up to and including this block is
   * stable across requests and SHOULD be cached.
   * 
   * OBSERVABLE BEHAVIOR:
   * If a subsequent request contains identical content up to
   * a cache-hinted block, the provider:
   * - SHOULD report `cached_tokens > 0` in usage
   * - SHOULD NOT re-process the cached content (cost reduction)
   * - MUST produce equivalent output to uncached execution
   * 
   * MULTIPLE HINTS:
   * Multiple `cache` hints in a request create multiple cache
   * breakpoints. Providers MAY limit the number of breakpoints.
   * Provider-specific limits are documented per-provider.
   */
  cache?: CacheHint;
}

// Similarly for:
// - InputImageContentParam
// - InputFileContentParam
```

### 4. Usage Reporting

```typescript
interface InputTokensDetails {
  /**
   * Number of input tokens read from cache.
   * 
   * NORMATIVE: Providers MUST report this field when cache hints
   * are provided and caching is supported, even if the value is 0.
   */
  cached_tokens?: number;
  
  /**
   * Number of input tokens written to cache.
   * 
   * Reported when new content is added to the cache. Subsequent
   * requests with the same content will report `cached_tokens`
   * instead.
   * 
   * NORMATIVE: Providers that support explicit caching SHOULD
   * report this field to indicate cache population.
   */
  cache_write_tokens?: number;
}
```

---

## Provider Mapping

### Anthropic Claude

**Caching Model**: Prefix-based breakpoints with content-addressed identity.

| OpenResponses Feature | Anthropic Mapping | Notes |
|----------------------|-------------------|-------|
| `cache.tools` | `cache_control` on last tool in array | Tools cached as atomic unit |
| `cache.instructions` | `cache_control` on system message | |
| Content `cache` | `cache_control` on content block | Direct mapping |
| `ttl: "ephemeral"` | Default (5 min TTL) | Within bounds |
| `ttl: "session"` | Extended (1 hour TTL) | Within bounds |
| `ttl: "extended"` | Extended (1 hour TTL) | Anthropic max is 1h |
| `cached_content` | ❌ Not supported | Error or ignore |
| `cached_tokens` | `cache_read_input_tokens` | |
| `cache_write_tokens` | `cache_creation_input_tokens` | |

**Request Translation Example**:

```typescript
// OpenResponses request
{
  instructions: "You are a helpful assistant.",
  cache: {
    instructions: { type: "ephemeral", ttl: "session" }
  },
  tools: [...],
  input: [
    { type: "input_text", text: "Context...", cache: { type: "ephemeral" } },
    { type: "input_text", text: "Question?" }
  ]
}

// Anthropic translation
{
  system: [
    {
      type: "text",
      text: "You are a helpful assistant.",
      cache_control: { type: "ephemeral" }  // With extended TTL header
    }
  ],
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Context...", cache_control: { type: "ephemeral" } },
        { type: "text", text: "Question?" }
      ]
    }
  ]
}
```

**Conformance Notes**:
- Maximum 4 cache breakpoints per request
- Minimum cacheable tokens: 1024 (Sonnet), 2048 (Haiku)
- Cache TTL: 5 minutes (ephemeral) or 1 hour (session/extended)

---

### Google Gemini

**Caching Model**: Named cache objects with explicit lifecycle.

| OpenResponses Feature | Gemini Mapping | Notes |
|----------------------|----------------|-------|
| `cache.tools` | Create/reuse tools cache object | Gateway manages lifecycle |
| `cache.instructions` | Create/reuse instructions cache object | Gateway manages lifecycle |
| Content `cache` | Include in content cache object | Aggregated, not breakpoints |
| `ttl: "ephemeral"` | TTL ≥ 5 minutes | |
| `ttl: "session"` | TTL ≥ 1 hour | |
| `ttl: "extended"` | TTL ≥ 4 hours | Gemini supports arbitrary TTL |
| `cached_content` | `cachedContent` reference | Direct mapping |
| `cached_tokens` | `cachedContentTokenCount` | |
| `cache_write_tokens` | From cache creation response | |

**Gateway Behavior**:

When `cache` hints are provided, the gateway:

1. **Computes content hash** of tools/instructions/hinted content
2. **Checks for existing cache** with matching hash
3. **Creates cache if needed** with appropriate TTL
4. **References cache** in Gemini request via `cachedContent`

```typescript
// OpenResponses request
{
  instructions: "You are a helpful assistant.",
  cache: {
    instructions: { type: "ephemeral", ttl: "session" }
  },
  input: [...]
}

// Gateway behavior (pseudocode)
const hash = computeHash(request.instructions);
let cache = await findCache(hash);
if (!cache) {
  cache = await gemini.caches.create({
    model: "gemini-1.5-flash",
    systemInstruction: request.instructions,
    ttl: "3600s"  // 1 hour for "session"
  });
}

// Gemini request
{
  cachedContent: cache.name,
  contents: translateInput(request.input)
}
```

**`cached_content` Support**:

Gemini is the primary target for `cached_content`. When provided:

```typescript
// OpenResponses request
{
  cached_content: "caches/abc123",
  input: [{ type: "input_text", text: "Follow-up question" }]
}

// Gemini request (direct mapping)
{
  cachedContent: "caches/abc123",
  contents: [{ role: "user", parts: [{ text: "Follow-up question" }] }]
}
```

**Conformance Notes**:
- Minimum cacheable tokens: 32,768
- Cache TTL: Configurable, can be updated
- Cache objects have names and can be listed/deleted

---

### OpenAI / Azure OpenAI

**Caching Model**: Implicit automatic caching, no client control.

| OpenResponses Feature | OpenAI Mapping | Notes |
|----------------------|----------------|-------|
| `cache.tools` | ❌ Ignored | No explicit control |
| `cache.instructions` | ❌ Ignored | No explicit control |
| Content `cache` | ❌ Ignored | No explicit control |
| `ttl` | ❌ Ignored | Provider-determined |
| `cached_content` | ❌ Error or ignored | Not supported |
| `cached_tokens` | `prompt_tokens_details.cached_tokens` | Automatic reporting |
| `cache_write_tokens` | N/A | Implicit, not reported |

**Gateway Behavior**:

Cache hints are stripped from the request. Usage is mapped from response:

```typescript
// OpenResponses usage
{
  input_tokens: 1000,
  input_tokens_details: {
    cached_tokens: response.usage.prompt_tokens_details?.cached_tokens ?? 0,
    cache_write_tokens: 0  // Not reported by OpenAI
  }
}
```

**Conformance Notes**:
- Caching is automatic based on prompt prefix matching
- No minimum token requirement documented
- No TTL control; provider-managed
- Cache hints are valid but have no effect

---

### Mistral / Cohere

**Caching Model**: Not documented as of 2026-02-04.

| OpenResponses Feature | Mapping | Notes |
|----------------------|---------|-------|
| All `cache` fields | ❌ Ignored | No caching support documented |
| `cached_tokens` | 0 | No cache hits possible |
| `cache_write_tokens` | 0 | No cache population |

**Gateway Behavior**:

Cache hints are stripped. Usage fields report 0 for cache-related metrics.

---

## Gap Analysis

### What OpenResponses Cache Hints CAN Express

| Capability | Supported | Notes |
|------------|-----------|-------|
| "Cache this content" | ✅ | Via `cache` hints |
| "Cache tools/instructions" | ✅ | Via `cache.tools`, `cache.instructions` |
| Request-scoped caching | ✅ | Hints apply to single request |
| Duration preferences | ✅ | Via `ttl` categories |
| Cache hit reporting | ✅ | Via `cached_tokens` |
| Cache write reporting | ✅ | Via `cache_write_tokens` |

### What OpenResponses Cache Hints CANNOT Express

| Capability | Gap For | Mitigation |
|------------|---------|------------|
| Named cache creation | Gemini | Use native API or `provider_options` |
| Cache listing/inspection | Gemini | Use native API |
| Cache deletion | Gemini | Use native API |
| Explicit cache lifecycle | Gemini | Use native API |
| Cache sharing across requests | Gemini | `cached_content` (reserved) |
| Per-tool caching | None (no provider supports) | N/A |
| Disable implicit caching | OpenAI (if needed) | Not expressible |

### Migration Path for `cached_content`

Applications requiring full Gemini cache lifecycle management have two options:

**Option 1: Use `provider_options` (Now)**

```typescript
{
  provider_options: {
    google: {
      cached_content: "caches/abc123"
    }
  },
  input: [...]
}
```

**Option 2: Use `cached_content` (When Standardized)**

```typescript
{
  cached_content: "caches/abc123",
  input: [...]
}
```

The reserved `cached_content` field provides a migration path. Gateways MAY implement it now; it will be standardized in a future spec revision once usage patterns are established.

---

## Conformance Requirements

### For Providers Supporting Explicit Caching (Anthropic, Gemini)

1. **MUST** report `cached_tokens` when cache hints are provided
2. **SHOULD** report `cache_write_tokens` when new content is cached
3. **MUST** cache for at least the minimum duration when `ttl` is honored
4. **MUST** produce equivalent output whether content is cached or not

### For Providers With Implicit Caching (OpenAI, Azure)

1. **SHOULD** report `cached_tokens` from automatic caching
2. **MAY** ignore all `cache` hints without error
3. **SHOULD** return error for `cached_content` (not supported)

### For Providers Without Caching (Mistral, Cohere)

1. **MAY** ignore all `cache` hints without error
2. **SHOULD** report `cached_tokens: 0` when hints are provided
3. **SHOULD** return error for `cached_content` (not supported)

---

## Test Vectors

### Test 1: Cache Hint Behavior

```typescript
// Verify cache hints result in cache hits on repeated requests

const content = [
  { type: "input_text", text: "Cached prefix", cache: { type: "ephemeral" } },
  { type: "input_text", text: "Uncached suffix" }
];

// Request 1: Expect cache miss (write)
const r1 = await createResponse({ input: content });
// For providers with explicit caching:
// - cache_write_tokens SHOULD be > 0
// - cached_tokens SHOULD be 0

// Request 2: Expect cache hit (read)  
const r2 = await createResponse({ input: content });
// For providers with explicit caching:
// - cached_tokens SHOULD be > 0
// - cache_write_tokens SHOULD be 0
```

### Test 2: TTL Minimum Duration

```typescript
// Verify cache persists for at least minimum duration

const content = [
  { type: "input_text", text: "Long-lived content", cache: { type: "ephemeral", ttl: "session" } }
];

const r1 = await createResponse({ input: content });

// Wait 30 minutes (session minimum)
await sleep(30 * 60 * 1000);

const r2 = await createResponse({ input: content });
// cached_tokens SHOULD be > 0 (cache still valid)
```

### Test 3: Tools Cache Hint

```typescript
// Verify tools caching works as a unit

const request = {
  tools: [
    { type: "function", name: "search", ... },
    { type: "function", name: "fetch", ... }
  ],
  cache: {
    tools: { type: "ephemeral" }
  },
  input: [{ type: "input_text", text: "Use search" }]
};

const r1 = await createResponse(request);
const r2 = await createResponse(request);

// r2.cached_tokens SHOULD include tokens from tools
```

---

## Open Issues

1. **Cache hint on `instructions` field**: The `instructions` field is a string, not an object. Should we support `cache.instructions` only at request level, or add a structured `instructions` variant?

2. **Maximum breakpoints**: Should the spec define a minimum number of breakpoints providers must support, or leave this to provider documentation?

3. **Cache key stability**: When using content-addressed caching (Anthropic), minor content changes invalidate the cache. Should we document this behavior?

4. **Error behavior for `cached_content`**: Should providers return an error or silently ignore when `cached_content` is provided but not supported?

---

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [Azure OpenAI Reference (Preview)](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/reference-preview)
- [OpenAI Responses API](https://platform.openai.com/docs/api-reference/responses)
- [CLDR/Intl Design Principles](https://tc39.es/ecma402/)
