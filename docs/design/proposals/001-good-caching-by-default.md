# Proposal 001: Good Caching by Default

**Status**: Draft  
**Date**: 2026-02-04  
**Authors**: Yehuda Katz  
**Urgency**: High — enables significant cost savings with minimal spec surface

## Abstract

This proposal adds a simple opt-in mechanism for automatic caching behavior
in OpenResponses. When enabled, gateways apply provider-appropriate caching
strategies without requiring clients to understand provider-specific details.

The design leverages the fact that most modern LLM providers already cache
content implicitly. For providers that require explicit hints (Anthropic),
gateways translate the opt-in into appropriate cache markers.

## Motivation

Prompt caching can reduce costs by up to 90% and significantly improve latency.
However, the current landscape is fragmented:

- **Anthropic**: Requires explicit `cache_control` markers; cache writes cost 25% more
- **Gemini**: Implicit caching is automatic; explicit caching requires separate API
- **OpenAI/Azure**: Implicit caching is automatic; no explicit control available

Clients building provider-agnostic applications face a choice:
1. Ignore caching entirely (leave money on the table)
2. Implement provider-specific logic (defeats the purpose of OpenResponses)

This proposal provides a third option: opt into "good caching behavior" and let
the gateway handle provider differences.

## Design

### The `caching` Field

```typescript
interface CreateResponseRequest {
  // ... existing fields ...
  
  /**
   * Caching behavior hint.
   * 
   * - `"auto"`: Gateway applies provider-appropriate caching to stable content
   * - `undefined` / omitted: No caching hints applied (provider default behavior)
   * 
   * When `"auto"` is specified, gateways SHOULD apply caching to:
   * - Tools (if present)
   * - Instructions/system message (if present)
   * - Conversation history prefix (content before the final user message)
   */
  caching?: "auto";
}
```

### What "Auto" Means

When `caching: "auto"` is specified, the gateway:

1. **Identifies stable content**: Tools, instructions, and conversation prefix
2. **Applies provider-appropriate caching**:
   - Anthropic: Adds `cache_control` markers to stable content
   - Gemini: No action needed (implicit caching handles it)
   - OpenAI: No action needed (implicit caching handles it)
3. **Reports cache effectiveness**: Via existing `cached_tokens` in usage

### Programming Model: Union of Limitations

Clients using `caching: "auto"` should program to the **union of provider
limitations** to ensure consistent behavior:

| Constraint | Source | Client Guidance |
|------------|--------|-----------------|
| **Content-addressed** | Anthropic | Same bytes = cache hit. Any change (even whitespace) invalidates the cache. |
| **Prefix-based** | All | Only content prefixes can be cached. Put stable content first. |
| **Order matters** | Anthropic | Cache is built in order: tools → system → messages. |
| **Minimum size** | Anthropic (1024 tokens), Gemini (32K) | Very small content may not be cached. |

### Example

```typescript
// Client code — works across all providers
const response = await client.responses.create({
  model: "gpt-4",
  caching: "auto",  // "Give me good caching behavior"
  instructions: "You are a helpful assistant specialized in...",
  tools: [...],
  input: [
    // Conversation history (stable, will be cached)
    { type: "input_text", text: "Previous context..." },
    { type: "input_text", text: "More history..." },
    // Current turn (not cached)
    { type: "input_text", text: "User's new question" }
  ]
});

// Check cache effectiveness (works on all providers)
console.log(`Cached tokens: ${response.usage.input_tokens_details?.cached_tokens}`);
```

## Gateway Behavior

### Anthropic

When `caching: "auto"`:

1. Add `cache_control: { type: "ephemeral" }` to:
   - The last tool in the tools array (caches all tools)
   - The system message (caches instructions)
   - The last message before the final user turn (caches conversation prefix)

2. Respect Anthropic's 4-breakpoint limit by prioritizing:
   - Tools (if present)
   - Instructions (if present)
   - Conversation prefix (remaining breakpoints)

```typescript
// Anthropic translation
{
  system: [{ 
    type: "text", 
    text: "You are...", 
    cache_control: { type: "ephemeral" }  // Added by gateway
  }],
  messages: [...]
}
```

### Gemini

When `caching: "auto"`:

1. **No action required** — Gemini's implicit caching handles repeated prefixes
2. Report `cached_tokens` from `usage_metadata.cached_content_token_count`

### OpenAI / Azure

When `caching: "auto"`:

1. **No action required** — OpenAI's implicit caching handles repeated prefixes
2. Report `cached_tokens` from `prompt_tokens_details.cached_tokens`

## Usage Reporting

All providers report cache effectiveness via the existing `cached_tokens` field:

```typescript
interface ResponseUsage {
  input_tokens: number;
  output_tokens: number;
  input_tokens_details?: {
    cached_tokens: number;  // Already in base spec
  };
}
```

## Non-Normative: When to Use `caching: "auto"`

> **Note:** This guidance is non-normative. Clients should evaluate their
> specific use case.

`caching: "auto"` is most beneficial for:

- **Multi-turn conversations**: Where the same context is sent repeatedly
- **Tool-heavy applications**: Where tool definitions are stable across requests
- **Long system prompts**: Where instructions don't change between requests

`caching: "auto"` may not be beneficial for:

- **One-off requests**: Cache writes have overhead (25% on Anthropic)
- **Highly dynamic content**: Where nothing is stable across requests

**Heuristic for generic clients:** Enable `caching: "auto"` when
`previous_response_id` is present (indicating a multi-turn conversation),
or when the application knows it will make repeated similar requests.

Since implicit caching is free on Gemini and OpenAI, the main consideration
is Anthropic's cache write cost. For multi-turn conversations, the cache
read savings (90%) far outweigh the initial write cost (25%).

## Conformance

### For Gateways

1. **MUST** support `caching: "auto"` field
2. **SHOULD** apply caching to stable content when `caching: "auto"` is specified
3. **MUST** report `cached_tokens` in usage when caching occurs
4. **MAY** ignore `caching: "auto"` if the provider has no caching mechanism

### For Clients

1. **SHOULD** program to the union of provider limitations when using `caching: "auto"`
2. **SHOULD NOT** rely on specific caching behavior beyond what's observable in `cached_tokens`

## Non-Goals (Deferred to Proposal 002)

This proposal intentionally excludes:

- Fine-grained cache hints on individual content parts
- TTL/duration control
- Named caches (`cached_content`)
- Cache write reporting (`cache_write_tokens`)

These features require more design work and provider feedback. They are
addressed in [Proposal 002: Cache Control Hints](./002-cache-control-hints.md).

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
