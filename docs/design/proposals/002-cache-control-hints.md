# Cache Control Hints

**Status**: Draft  
**Authors**: Yehuda Katz  
**Depends On**: [001-good-caching-by-default.md](./001-good-caching-by-default.md)

## Summary

Extend `caching: "auto"` with fine-grained control: TTL hints, content-level
breakpoints, named cache references, and cache write reporting.

## Motivation

Proposal 001 covers the common case. Power users need:

- **TTL control** — session-scoped vs ephemeral caching
- **Precise breakpoints** — cache specific content, not everything
- **Named caches** — reuse across requests (Gemini model)
- **Write observability** — distinguish cache hits from writes

## Specification

### Extended `caching` Field

```typescript
interface CreateResponseRequest {
  caching?: "auto" | CacheConfig;
  cached_content?: string;  // Named cache reference
}

interface CacheConfig {
  tools?: CacheHint;
  instructions?: CacheHint;
}

interface CacheHint {
  /**
   * | Category | Minimum | Use Case                    |
   * |----------|---------|------------------------------|
   * | "short"  | 1 min   | Single interaction           |
   * | "medium" | 30 min  | Multi-turn conversation      |
   * | "long"   | 4 hours | Cross-session reuse          |
   */
  ttl?: "short" | "medium" | "long";
}
```

### Content-Level Hints

```typescript
interface InputTextContentParam {
  type: "input_text";
  text: string;
  cache?: CacheHint;  // Marks breakpoint
}
// Similarly: InputImageContentParam, InputFileContentParam
```

### Cache Write Reporting

```typescript
interface InputTokensDetails {
  cached_tokens: number;
  cache_write_tokens?: number;  // NEW
}
```

## Provider Mapping

### Anthropic

| Feature | Translation |
|---------|-------------|
| `caching.tools` | `cache_control` on last tool |
| `caching.instructions` | `cache_control` on system message |
| Content `cache` | `cache_control` on block |
| `ttl: "short"` | Default (5 min) |
| `ttl: "medium"/"long"` | Extended (1 hour) |
| `cached_content` | Error (unsupported) |
| `cache_write_tokens` | `cache_creation_input_tokens` |

### Gemini

| Feature | Translation |
|---------|-------------|
| `caching.*` | Create cache objects |
| `ttl` | Direct TTL mapping |
| `cached_content` | `cachedContent` field |
| `cache_write_tokens` | From creation response |

### OpenAI / Azure

| Feature | Translation |
|---------|-------------|
| All hints | Ignored (implicit only) |
| `cached_content` | Error (unsupported) |

## Conformance

**Explicit caching providers (Anthropic, Gemini):**
- MUST honor TTL minimums
- MUST report `cached_tokens`
- SHOULD report `cache_write_tokens`

**Implicit caching providers (OpenAI, Azure):**
- MAY ignore hints without error
- MUST error on `cached_content`

**No-caching providers:**
- MAY ignore hints without error
- MUST error on `cached_content`

## Relationship to Proposal 001

`caching: "auto"` is shorthand for:

```typescript
caching: {
  tools: {},
  instructions: {}
}
```

Plus automatic conversation prefix caching.

## Open Questions

1. TTL interaction with `cached_content` — which wins?
2. Minimum supported breakpoints — should spec define?
3. Cache key stability — document invalidation behavior?

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
