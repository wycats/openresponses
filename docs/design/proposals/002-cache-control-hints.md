# Cache Control Hints

**Status**: Draft  
**Authors**: Yehuda Katz  
**Depends On**: [001-good-caching-by-default.md](./001-good-caching-by-default.md)

## Summary

Extend `caching: "auto"` with fine-grained control: TTL hints, content-level
breakpoints, named cache references, and cache write reporting.

```typescript
interface CreateResponseRequest {
  caching?: "auto" | CacheConfig;
  cached_content?: string;
}
```

This proposal follows the same design philosophy as Proposal 001: clients write
portable code, gateways translate to provider-specific behavior.

## Motivation

While Proposal 001's `caching: "auto"` covers the common case, some applications
need finer control: TTL selection (session-length vs brief), precise breakpoints
(cache exactly this content), named caches for cross-request reuse, or write
observability (distinguish cache hits from population).

This proposal provides these capabilities while maintaining portability.

## Specification

### Extended `caching` Field

```typescript
interface CreateResponseRequest {
  caching?: "auto" | CacheConfig;
  
  /**
   * Reference to a named cache object (provider-specific).
   * 
   * On providers that support named caches (Gemini), the cache content
   * acts as an implicit prefix. On providers that don't support named
   * caches, this field is silently ignored.
   */
  cached_content?: string;
}

interface CacheConfig {
  tools?: CacheHint;
  instructions?: CacheHint;
}

interface CacheHint {
  /**
   * Duration hint for cached content.
   * 
   * | Category | Minimum | Use Case                    |
   * |----------|---------|------------------------------|
   * | "short"  | 1 min   | Single interaction           |
   * | "medium" | 30 min  | Multi-turn conversation      |
   * | "long"   | 4 hours | Cross-session reuse          |
   * 
   * Providers that honor TTL hints MUST cache for at least the minimum
   * duration. Providers MAY cache longer.
   */
  ttl?: "short" | "medium" | "long";
}
```

### Content-Level Hints

Content parts can include cache hints to mark explicit breakpoints:

```typescript
interface InputTextContentParam {
  type: "input_text";
  text: string;
  cache?: CacheHint;  // Marks this as a cache breakpoint
}
// Similarly: InputImageContentParam, InputFileContentParam
```

Multiple content-level hints create multiple breakpoints, subject to provider
limits (e.g., Anthropic's 4-breakpoint maximum).

### Cache Write Reporting

```typescript
interface InputTokensDetails {
  cached_tokens: number;       // Existing: tokens read from cache
  cache_write_tokens?: number; // NEW: tokens written to cache
}
```

This enables clients to distinguish cache hits (`cached_tokens > 0`) from
cache population (`cache_write_tokens > 0`).

## Provider Mapping

### Anthropic

| Feature | Translation |
|---------|-------------|
| `caching.tools` | `cache_control` on last tool |
| `caching.instructions` | `cache_control` on system message |
| Content `cache` | `cache_control` on content block |
| `ttl: "short"` | Default TTL (5 min) |
| `ttl: "medium"` / `"long"` | Extended TTL (1 hour) |
| `cached_content` | Silently ignored (not supported) |
| `cache_write_tokens` | Maps to `cache_creation_input_tokens` |

### Gemini

| Feature | Translation |
|---------|-------------|
| `caching.*` | Create/reuse cache objects |
| `ttl` | Direct TTL mapping (≥ minimum) |
| `cached_content` | Maps to `cachedContent` field |
| `cache_write_tokens` | From cache creation response |

### OpenAI / Azure

| Feature | Translation |
|---------|-------------|
| All `caching` hints | Silently ignored (implicit caching only) |
| `cached_content` | Silently ignored (not supported) |
| `cache_write_tokens` | Not reported |

## Conformance

Gateways:

- MUST accept all fields without error
- SHOULD apply hints when the provider supports them
- MAY silently ignore hints the provider doesn't support
- MUST report `cached_tokens` when available
- SHOULD report `cache_write_tokens` when available

Clients:

- SHOULD NOT depend on hints being honored
- SHOULD use `cached_tokens` / `cache_write_tokens` for observability
- MAY use `cached_content` knowing it's provider-specific

## Relationship to Proposal 001

`caching: "auto"` remains the recommended default. It is equivalent to:

```typescript
caching: {
  tools: {},
  instructions: {}
}
```

Plus automatic conversation prefix caching.

Clients should start with `caching: "auto"` and only use fine-grained control
when they have specific requirements that justify the complexity.

## Open Questions

1. If both `cached_content` and `caching` specify TTL, which wins?
2. Should the spec define a minimum number of breakpoints gateways must support?
3. Should we document content-addressed cache invalidation behavior?

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [Cache Control Explainer](./explainer-cache-control.md)
