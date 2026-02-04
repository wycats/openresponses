# Proposal 002: Cache Control Hints

**Status**: Draft  
**Date**: 2026-02-04  
**Authors**: Yehuda Katz  
**Depends On**: [Proposal 001: Good Caching by Default](./001-good-caching-by-default.md)  
**Urgency**: Medium — benefits from provider feedback and implementation experience

## Abstract

This proposal extends the caching capabilities introduced in Proposal 001 with
fine-grained control over caching behavior. It adds:

1. **TTL hints** with semantic duration categories
2. **Content-level cache hints** for precise breakpoint control
3. **Named cache references** for providers with explicit cache management
4. **Cache write reporting** for observability

The design follows the CLDR/Intl pattern: semantic categories with normative
bounds, allowing providers flexibility while ensuring predictable behavior.

## Motivation

While Proposal 001's `caching: "auto"` covers the common case, power users need:

- **TTL control**: "Keep this cached for the session" vs "cache briefly"
- **Precise breakpoints**: "Cache exactly this content, not that"
- **Named caches**: Reuse caches across requests (Gemini's model)
- **Write observability**: Know when cache was populated vs hit

This proposal provides these capabilities while maintaining the Intl-style
design philosophy: semantic intent over literal values, with normative bounds.

## Design

### Extending the `caching` Field

```typescript
interface CreateResponseRequest {
  // From Proposal 001
  caching?: "auto" | CacheConfig;
}

interface CacheConfig {
  /**
   * Cache hints for request-level components.
   */
  tools?: CacheHint;
  instructions?: CacheHint;
}

interface CacheHint {
  /**
   * Duration category for cached content.
   * 
   * | Category | Minimum | Description                              |
   * |----------|---------|------------------------------------------|
   * | "short"  | 1 min   | Very short-lived, single interaction     |
   * | "medium" | 30 min  | Session-scoped, multi-turn conversation  |
   * | "long"   | 4 hours | Long-lived, cross-session reuse          |
   * 
   * Default: "short"
   * 
   * NORMATIVE: Providers that honor this hint MUST cache for at least
   * the specified minimum duration.
   */
  ttl?: "short" | "medium" | "long";
}
```

### Content-Level Cache Hints

```typescript
interface InputTextContentParam {
  type: "input_text";
  text: string;
  
  /**
   * Cache hint for this content block.
   * 
   * Indicates that content up to and including this block is stable
   * and SHOULD be cached. Multiple hints create multiple breakpoints.
   */
  cache?: CacheHint;
}

// Similarly for InputImageContentParam, InputFileContentParam
```

### Named Cache References

```typescript
interface CreateResponseRequest {
  /**
   * Reference to a named cache object.
   * 
   * When specified, the cache content acts as an implicit prefix.
   * The cache MUST exist and be accessible.
   * 
   * Providers that do not support named caches MUST return an error.
   */
  cached_content?: string;
}
```

### Cache Write Reporting

```typescript
interface InputTokensDetails {
  cached_tokens: number;      // From base spec
  cache_write_tokens?: number; // NEW: tokens written to cache
}
```

## Provider Mapping

### Anthropic

| Feature | Mapping |
|---------|---------|
| `caching.tools` | `cache_control` on last tool |
| `caching.instructions` | `cache_control` on system message |
| Content `cache` | `cache_control` on content block |
| `ttl: "short"` | Default (5 min) |
| `ttl: "medium"` | Extended (1 hour) |
| `ttl: "long"` | Extended (1 hour) — Anthropic max |
| `cached_content` | **Error** — not supported |
| `cache_write_tokens` | `cache_creation_input_tokens` |

### Gemini

| Feature | Mapping |
|---------|---------|
| `caching.tools` | Create/reuse tools cache object |
| `caching.instructions` | Create/reuse instructions cache object |
| Content `cache` | Include in content cache object |
| `ttl: "short"` | TTL ≥ 5 minutes |
| `ttl: "medium"` | TTL ≥ 1 hour |
| `ttl: "long"` | TTL ≥ 4 hours |
| `cached_content` | Direct mapping to `cachedContent` |
| `cache_write_tokens` | From cache creation response |

**Gateway Options:**
1. **Implicit caching** (simple): Ignore hints, rely on Gemini's automatic caching
2. **Explicit caching** (full): Create cache objects based on hints

### OpenAI / Azure

| Feature | Mapping |
|---------|---------|
| All `caching` hints | Ignored (implicit caching only) |
| `cached_content` | **Error** — not supported |
| `cache_write_tokens` | Not reported (implicit) |

## Intl-Style Design Principles

### 1. Semantic Categories with Normative Bounds

TTL values are semantic (`"short"`, `"medium"`, `"long"`), not literal durations.
Each category has a **normative minimum** that providers MUST honor.

This allows:
- Anthropic to map `"medium"` → 1 hour (their extended TTL)
- Gemini to map `"medium"` → exactly 1 hour or longer
- Future providers to choose appropriate values within bounds

### 2. Observable Behavior Contracts

The spec defines behavior in terms of **observable outcomes**:
- `cached_tokens > 0` indicates cache hit
- `cache_write_tokens > 0` indicates cache population
- Providers MUST produce equivalent output whether cached or not

### 3. Provider Escape Hatches

For provider-specific features not covered by this spec:

```typescript
{
  caching: { tools: { ttl: "medium" } },
  provider_options: {
    anthropic: {
      // Anthropic-specific cache options
    },
    google: {
      // Gemini-specific cache options (e.g., cache name)
    }
  }
}
```

## Conformance

### For Providers Supporting Explicit Caching (Anthropic, Gemini)

1. **MUST** honor TTL minimums when hints are provided
2. **MUST** report `cached_tokens` when caching occurs
3. **SHOULD** report `cache_write_tokens` when cache is populated
4. **MUST** produce equivalent output whether cached or not

### For Providers With Implicit Caching (OpenAI, Azure)

1. **MAY** ignore all cache hints without error
2. **SHOULD** report `cached_tokens` from automatic caching
3. **MUST** return error for `cached_content` (not supported)

### For Providers Without Caching

1. **MAY** ignore all cache hints without error
2. **MUST** return error for `cached_content` (not supported)

## Migration from Proposal 001

Proposal 001's `caching: "auto"` remains valid and is equivalent to:

```typescript
// caching: "auto" is shorthand for:
caching: {
  tools: {},
  instructions: {}
}
// Plus automatic conversation prefix caching
```

Clients can start with `caching: "auto"` and migrate to fine-grained control
as needed.

## Open Questions

1. **TTL interaction with `cached_content`**: If both are specified, which wins?
2. **Maximum breakpoints**: Should spec define minimum supported breakpoints?
3. **Cache key stability**: Document content-addressed invalidation behavior?

## References

- [Proposal 001: Good Caching by Default](./001-good-caching-by-default.md)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [CLDR/Intl Design Principles](https://tc39.es/ecma402/)
