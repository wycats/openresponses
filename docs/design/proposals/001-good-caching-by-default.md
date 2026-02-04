# Automatic Caching

**Status**: Draft  
**Authors**: Yehuda Katz

## Summary

Add a `caching` field to `CreateResponseRequest` that enables automatic,
provider-appropriate caching of stable request content.

## Motivation

Prompt caching reduces costs (up to 90% on cache hits) and latency. Provider
implementations differ:

| Provider | Mechanism | Client Action Required |
|----------|-----------|------------------------|
| Anthropic | Explicit breakpoints | Yes (`cache_control` markers) |
| Gemini | Implicit (automatic) | No |
| OpenAI | Implicit (automatic) | No |

Clients targeting multiple providers must either implement provider-specific
caching logic or forgo caching entirely. This proposal enables portable caching
with a single field.

## Specification

### Schema

```typescript
interface CreateResponseRequest {
  /**
   * When "auto", gateways apply provider-appropriate caching to stable
   * content (tools, instructions, conversation prefix).
   */
  caching?: "auto";
}
```

### Stable Content

Gateways identify stable content as:

1. **Tools** — The entire tools array
2. **Instructions** — The system message
3. **Conversation prefix** — All input items except the final user message

### Gateway Translation

**Anthropic**: Add `cache_control: { type: "ephemeral" }` to stable content,
respecting the 4-breakpoint limit.

**Gemini, OpenAI**: No translation required. Implicit caching activates
automatically for repeated prefixes.

### Usage Reporting

Gateways report cache hits via the existing `cached_tokens` field:

```typescript
response.usage.input_tokens_details?.cached_tokens
```

## Client Programming Model

Clients using `caching: "auto"` should assume the union of provider constraints:

| Constraint | Implication |
|------------|-------------|
| Content-addressed | Byte-identical content required for cache hit |
| Prefix-based | Only leading content is cached; place stable content first |
| Ordered | Tools cached before instructions before messages |
| Minimum size | Content below ~1K tokens may not cache |

## Conformance

**Gateways:**
- MUST accept `caching: "auto"` without error
- SHOULD apply caching to stable content when specified
- MUST report `cached_tokens` when caching occurs

**Clients:**
- SHOULD NOT depend on caching behavior beyond `cached_tokens` reporting

## Non-Normative Notes

### When to Enable

`caching: "auto"` benefits multi-turn conversations and applications with
stable tools or instructions. For single requests with no reuse, the cache
write overhead on some providers (25% on Anthropic) may exceed benefits.

A reasonable heuristic: enable when `previous_response_id` is present.

### Cost Model

On providers with implicit caching (Gemini, OpenAI), `caching: "auto"` has
no cost impact. On Anthropic, cache writes cost 25% more than regular input;
cache reads cost 90% less. Multi-turn conversations benefit significantly.

## Future Work

Fine-grained cache control (TTL hints, content-level breakpoints, named caches)
is deferred to a separate proposal pending implementation experience.

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
