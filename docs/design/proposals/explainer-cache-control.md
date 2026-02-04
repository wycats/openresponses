# Cache Control for OpenResponses

## Authors

- Yehuda Katz ([@wycats](https://github.com/wycats))

## Participate

- [OpenResponses Issue Tracker](https://github.com/openresponses/openresponses/issues)
- [Proposal Discussion](https://github.com/openresponses/openresponses/discussions)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Proposed Approach](#proposed-approach)
- [Alternatives Considered](#alternatives-considered)
- [Privacy and Security Considerations](#privacy-and-security-considerations)
- [Stakeholder Feedback](#stakeholder-feedback)
- [References](#references)
<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

LLM prompt caching can reduce API costs by up to 90%. However, each provider
implements caching differently: Anthropic requires explicit markers, Gemini
uses named cache objects, and OpenAI caches implicitly. This proposal adds
a portable `caching` field to OpenResponses that works across providers.

The key insight is that most applications share a common pattern—tools,
instructions, and conversation history are reused across turns—and this
pattern can be cached without provider-specific code.

## User-Facing Problem

Consider a developer building an AI coding assistant. The assistant uses
a fixed set of tools (file operations, terminal access, search) and a
consistent system prompt. In a typical session, the user might have 50+
conversation turns.

Without caching, the developer pays full price to re-encode those tools
and the system prompt on every single turn. With Anthropic, this overhead
can exceed the cost of the actual completion.

The developer wants to enable caching, but faces a problem:

```typescript
// ❌ Anthropic requires cache_control markers
tools: [
  { name: "read_file", ..., cache_control: { type: "ephemeral" } }
]

// ❌ Gemini uses a completely different API
await caches.create({ model: "...", contents: [...] })
const response = await model.generateContent({
  cachedContent: cache.name, ...
})

// ❌ OpenAI has no explicit caching API at all
// (but does cache automatically if your prefix is long enough)
```

The developer must now:

1. Detect which provider is in use
2. Implement provider-specific caching logic
3. Test across all providers
4. Maintain divergent code paths

Most developers choose "no caching" and accept the costs.

## Goals

- **Portable caching**: A single `caching: "auto"` field that works on
  Anthropic, Gemini, and OpenAI without provider-specific code.

- **Zero-config for common cases**: Multi-turn conversations with stable
  tools and instructions should "just work" without configuration.

- **Observability**: Report cache hits via `cached_tokens` so developers
  can verify caching is working.

- **Extensibility**: Support fine-grained control (TTL hints, named caches)
  for applications that need it.

## Non-goals

- **Guaranteed caching**: We cannot force providers to cache content. The
  spec defines hints that providers should honor, not mandates they must.

- **Identical behavior across providers**: Providers have different minimum
  sizes, TTLs, and pricing. A cache hit on one provider may be a miss on
  another.

- **Cross-request cache sharing by default**: Named caches (Gemini's model)
  are exposed but not required for basic functionality.

- **Cache invalidation API**: Explicit cache clearing is out of scope for
  this proposal.

## Proposed Approach

### Automatic Caching

The simplest form adds a single field:

```typescript
const response = await client.responses.create({
  model: "claude-sonnet-4-20250514",
  input: [...],
  tools: [...],
  instructions: "You are a helpful assistant.",
  caching: "auto"  // ← Enable portable caching
});
```

When `caching: "auto"` is set, the gateway:

1. Identifies stable content (tools, instructions, conversation prefix)
2. Applies provider-appropriate caching markers
3. Reports cache hits via `response.usage.input_tokens_details.cached_tokens`

The developer writes one code path. The gateway handles provider differences.

### What Gets Cached

The gateway identifies "stable content" using a simple heuristic:

| Content | Rationale |
|---------|-----------|
| Tools | Typically fixed for the application |
| Instructions | Typically fixed for the session |
| All messages except the last | Growing conversation history |

This matches the natural structure of multi-turn conversations.

### Provider Translation

**Anthropic**: The gateway adds `cache_control: { type: "ephemeral" }` to
stable content, respecting Anthropic's required order (tools → instructions
→ messages) and breakpoint limits.

**Gemini**: No translation needed. Gemini caches repeated prefixes automatically.

**OpenAI**: No translation needed. OpenAI caches prefixes ≥1024 tokens automatically.

### Fine-Grained Control

For applications needing more control, `caching` accepts an object:

```typescript
caching: {
  tools: { ttl: "long" },       // Keep tools cached across sessions
  instructions: { ttl: "medium" } // Keep instructions cached for the session
}
```

TTL categories use semantic names with normative minimum durations:

| Category | Minimum Duration | Use Case |
|----------|------------------|----------|
| `"short"` | 1 minute | Single interaction |
| `"medium"` | 30 minutes | Multi-turn conversation |
| `"long"` | 4 hours | Cross-session reuse |

Providers must cache for at least the minimum duration. They may cache longer.

### Content-Level Breakpoints

Individual content parts can mark explicit cache breakpoints:

```typescript
input: [
  {
    type: "message",
    role: "user",
    content: [
      {
        type: "input_text",
        text: longDocumentText,
        cache: {}  // ← Cache up to this point
      }
    ]
  }
]
```

### Named Caches (Gemini)

Gemini's named cache API is exposed but not required:

```typescript
// Create a named cache (Gemini-specific)
const cache = await caches.create({ model: "...", contents: [...] });

// Reference it
const response = await client.responses.create({
  cached_content: cache.name,  // Silently ignored on other providers
  input: [...]
});
```

The `cached_content` field is silently ignored on providers that don't
support it. This allows Gemini-optimized code without breaking portability.

### Cache Write Reporting

A new field reports when the cache was populated:

```typescript
response.usage.input_tokens_details.cache_write_tokens
```

Combined with `cached_tokens`, this enables full observability:

- `cached_tokens > 0, cache_write_tokens === 0`: Cache hit
- `cached_tokens === 0, cache_write_tokens > 0`: Cache miss, now cached
- Both zero: No caching occurred

## Alternatives Considered

### Per-Tool Cache Control

An earlier design allowed `cache_control` on individual tools:

```typescript
// ❌ Rejected
tools: [
  { name: "read_file", cache_control: { type: "ephemeral" } },
  { name: "write_file", cache_control: { type: "ephemeral" } }
]
```

This was rejected because Anthropic caches tools as a unit—you can't cache
some tools and not others. The per-tool markers created a false impression
of granularity.

### Literal TTL Values

An earlier design used numeric TTL values:

```typescript
// ❌ Rejected
caching: { tools: { ttl: 3600 } }  // seconds? milliseconds? provider-specific?
```

This was rejected because:
1. It exposed provider implementation details (Anthropic's 5m/1h choices)
2. Different providers have different minimum/maximum TTLs
3. Semantic categories (`"short"/"medium"/"long"`) are more portable

### Error on Unsupported Features

An earlier design had `cached_content` throw an error on unsupported providers:

```typescript
// ❌ Rejected
cached_content: "cache-123"  // Throws on Anthropic/OpenAI
```

This was rejected because it forces developers to write provider detection
code, defeating the portability goal. Silent ignore enables graceful
degradation.

### Explicit Provider Detection

We considered requiring developers to check provider support:

```typescript
// ❌ Rejected
if (gateway.supports("named_caches")) {
  const cache = await caches.create(...);
  response = await client.responses.create({ cached_content: cache.name, ... });
} else {
  response = await client.responses.create({ caching: "auto", ... });
}
```

This was rejected because it recreates the problem we're solving. The whole
point is to avoid provider-specific branches.

## Privacy and Security Considerations

**Cache Isolation**: Caches should be isolated per-session or per-user to
prevent cross-user data leakage. Gateways implementing named caches must
enforce appropriate isolation boundaries.

**Sensitive Content**: Developers should consider whether cached content
contains sensitive data. Cached content persists according to the TTL,
even if the original request is complete.

**Cost Visibility**: Cache write costs (25% extra on Anthropic) should be
clearly reported so developers can make informed decisions.

## Stakeholder Feedback

This proposal has not yet been formally reviewed by LLM providers or the
OpenResponses maintainers. Initial design was informed by:

- [Shaper's issue on provider options for caching](https://github.com/openresponses/openresponses/issues/XXX)
- Production experience with Anthropic caching in VS Code AI Gateway
- Analysis of OpenRouter and LiteLLM caching approaches

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
- [Proposal 001: Automatic Caching](./001-good-caching-by-default.md)
- [Proposal 002: Cache Control Hints](./002-cache-control-hints.md)
