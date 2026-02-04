# Cache Control Explainer

This document explains the design philosophy behind the OpenResponses cache
control extension. It accompanies the normative proposals:

- [Proposal 001: Automatic Caching](./001-good-caching-by-default.md)
- [Proposal 002: Cache Control Hints](./002-cache-control-hints.md)

## The Problem

LLM providers implement prompt caching differently:

| Provider | Model | Client Responsibility |
|----------|-------|----------------------|
| Anthropic | Explicit breakpoints | Mark cacheable content with `cache_control` |
| Gemini | Named cache objects | Create and manage cache objects via API |
| OpenAI | Implicit (automatic) | None—caching happens transparently |

A client targeting multiple providers faces a choice:

1. **Provider-specific code**: Implement different caching logic per provider
2. **No caching**: Ignore caching entirely, accept higher costs
3. **Lowest common denominator**: Only use features available everywhere (nothing)

None of these options is satisfactory.

## Design Philosophy

### The Intl/CLDR Pattern

The cache control extension follows the design pattern established by
JavaScript's `Intl` API and the Unicode CLDR:

> **Semantic categories with normative bounds**

Instead of exposing raw provider values, we define semantic categories that
express *intent*. Each category has normative bounds that providers must honor.

**Example: TTL**

Rather than exposing Anthropic's 5-minute and 1-hour TTLs directly:

```typescript
// ❌ Provider-specific
ttl: 300  // seconds? minutes? provider-specific?
```

We define semantic categories:

```typescript
// ✅ Semantic with normative bounds
ttl: "short"   // minimum 1 minute
ttl: "medium"  // minimum 30 minutes  
ttl: "long"    // minimum 4 hours
```

This allows:
- Anthropic to map `"medium"` → 1 hour (their extended TTL)
- Gemini to map `"medium"` → exactly 30 minutes or longer
- Future providers to choose appropriate values within bounds

### The Union of Limitations

When designing portable APIs, we consider the **union of limitations** across
providers. Clients who follow these constraints get good behavior everywhere:

| Constraint | Source | Implication |
|------------|--------|-------------|
| Content-addressed | All | Byte-identical content required for cache hit |
| Prefix-based | All | Only leading content is cached |
| Ordered | Anthropic | Tools → instructions → messages |
| Minimum size | OpenAI | Content below ~1K tokens may not cache |
| Breakpoint limit | Anthropic | Maximum 4 explicit breakpoints |

A client following all constraints gets optimal caching on every provider.

### Observable Behavior Contracts

The spec defines behavior in terms of **observable outcomes**, not
implementation details:

- `cached_tokens > 0` → cache hit occurred
- `cache_write_tokens > 0` → cache was populated
- Output equivalence → cached and uncached requests produce identical results

This allows providers flexibility in implementation while guaranteeing
predictable client behavior.

## The Two-Proposal Structure

### Why Two Proposals?

**Proposal 001** (Automatic Caching) addresses the 90% case:

- Single field: `caching: "auto"`
- Zero configuration
- Gateway handles all provider translation
- Ship immediately, benefit immediately

**Proposal 002** (Cache Control Hints) addresses power users:

- TTL control for different caching strategies
- Content-level breakpoints for precise control
- Named cache references for Gemini's model
- Cache write reporting for observability

This separation allows:
1. Quick adoption of basic caching (001)
2. Deliberate design of advanced features (002)
3. Implementation experience before committing to complex features

### The "Good Caching by Default" Axiom

The core design principle:

> A client using `caching: "auto"` with a well-structured request should get
> good caching behavior on any provider, without provider-specific code.

"Good" means:
- Cache hits when content is reused
- Reasonable TTL for the use case
- No unexpected costs or failures

This axiom drives all design decisions. Features that would violate it
(like requiring `cached_content` for basic caching) are rejected.

## Provider Landscape

### Anthropic: The Breakpoint Model

Anthropic requires explicit `cache_control` markers. Content is cached from
the start of the request up to each marker. Key constraints:

- Maximum 4 breakpoints per request
- Tools must be cached before system, system before messages
- Two TTL options: 5 minutes (default) or 1 hour (extended)
- Cache writes cost 25% more; cache reads cost 90% less

### Gemini: The Object Model

Gemini uses named cache objects that exist independently of requests:

- Create a cache object with content and TTL
- Reference it in requests via `cachedContent`
- Also supports implicit caching for repeated prefixes

This model enables cross-request cache sharing but requires explicit
cache management.

### OpenAI: The Implicit Model

OpenAI caches automatically with no client action:

- Prefixes ≥1024 tokens are cached
- 50% discount on cached tokens
- No explicit control available

This is the simplest model but offers no tuning.

## Future Considerations

### Provider Options Escape Hatch

For provider-specific features not covered by the spec:

```typescript
{
  caching: { tools: { ttl: "medium" } },
  provider_options: {
    anthropic: { /* Anthropic-specific */ },
    google: { /* Gemini-specific */ }
  }
}
```

This preserves portability for common cases while allowing provider-specific
optimization when needed.

### Potential Extensions

- **Cache statistics**: Hit rate, eviction count, etc.
- **Cache invalidation**: Explicit cache clearing
- **Cache sharing**: Cross-session or cross-user caching
- **Cost estimation**: Predict caching costs before request

These are deferred pending implementation experience with the core proposals.
