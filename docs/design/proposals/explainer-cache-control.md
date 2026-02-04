# Cache Control for OpenResponses

## Authors

- Yehuda Katz ([@wycats](https://github.com/wycats))

## Participate

- [OpenResponses Issue Tracker](https://github.com/openresponses/openresponses/issues)
- [Proposal Discussion](https://github.com/openresponses/openresponses/discussions)

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

- A single `caching: "auto"` field that works on Anthropic, Gemini, and
  OpenAI without provider-specific code.

- Multi-turn conversations with stable tools and instructions should
  benefit from caching without configuration.

- Cache hits should be reported via `cached_tokens` so developers can
  verify caching is working.

- Fine-grained control (TTL hints, named caches) should be available for
  applications that need it.

## Non-goals

- We cannot force providers to cache content. The spec defines hints that
  providers should honor, not mandates they must.

- Providers have different minimum sizes, TTLs, and pricing. A cache hit
  on one provider may be a miss on another. We do not aim for identical
  behavior.

- Named caches (Gemini's model) are exposed but not required for basic
  functionality.

- Explicit cache clearing is out of scope.

## Proposed Approach

### Automatic Caching

The simplest form adds a single field:

```typescript
const response = await client.responses.create({
  model: "claude-sonnet-4-20250514",
  input: [...],
  tools: [...],
  instructions: "You are a helpful assistant.",
  caching: "auto"
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
  tools: { ttl: "long" },
  instructions: { ttl: "medium" }
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
        cache: {}
      }
    ]
  }
]
```

### Named Caches (Gemini)

Gemini's named cache API is exposed but not required:

```typescript
const cache = await caches.create({ model: "...", contents: [...] });

const response = await client.responses.create({
  cached_content: cache.name,
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
caching: { tools: { ttl: 3600 } }
```

This was rejected because it exposed provider implementation details
(Anthropic's 5m/1h choices), different providers have different min/max
TTLs, and semantic categories are more portable.

### Error on Unsupported Features

An earlier design had `cached_content` throw an error on unsupported providers.
This was rejected because it forces developers to write provider detection
code, defeating the portability goal.

### Explicit Provider Detection

We considered requiring developers to check provider support:

```typescript
if (gateway.supports("named_caches")) {
  const cache = await caches.create(...);
  response = await client.responses.create({ cached_content: cache.name, ... });
} else {
  response = await client.responses.create({ caching: "auto", ... });
}
```

This recreates the problem we're solving.

## Privacy and Security Considerations

Caches should be isolated per-session or per-user to prevent cross-user
data leakage. Gateways implementing named caches must enforce appropriate
isolation boundaries.

Developers should consider whether cached content contains sensitive data.
Cached content persists according to the TTL, even after the original
request completes.

Cache write costs (25% extra on Anthropic) should be clearly reported so
developers can make informed decisions.

## Stakeholder Feedback

This proposal has not yet been formally reviewed. Initial design was informed
by production experience with Anthropic caching in [VS Code AI Gateway](https://github.com/vercel-labs/vscode-ai-gateway)
and analysis of [OpenRouter](https://openrouter.ai/docs/features/prompt-caching)
and [LiteLLM](https://docs.litellm.ai/docs/completion/prompt_caching) caching approaches.

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
- [Proposal 001: Automatic Caching](./001-good-caching-by-default.md)
- [Proposal 002: Cache Control Hints](./002-cache-control-hints.md)
