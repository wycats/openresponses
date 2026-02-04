# Automatic Caching

**Status**: Draft  
**Authors**: Yehuda Katz

## Summary

Add a `caching` field to `CreateResponseRequest` that enables automatic,
provider-appropriate caching of stable request content.

```typescript
interface CreateResponseRequest {
  caching?: "auto";
}
```

When set to `"auto"`, gateways apply provider-appropriate caching to stable
content (tools, instructions, conversation prefix). Clients get good caching
behavior without provider-specific code.

## Motivation

Prompt caching significantly reduces costs and latency—up to 90% savings on
cache hits. However, provider implementations differ substantially:

| Provider | Mechanism | Client Action Required |
|----------|-----------|------------------------|
| Anthropic | Explicit breakpoints | Yes (`cache_control` markers) |
| Gemini | Implicit (automatic) | No |
| OpenAI | Implicit (automatic) | No |

Clients targeting multiple providers must either implement provider-specific
caching logic or forgo caching entirely.

This proposal enables portable caching with a single field. A client can write
`caching: "auto"` and trust that the gateway will do the right thing for each
provider.

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

### What Gets Cached

Gateways identify "stable content" as:

| Content | Scope |
|---------|-------|
| Tools | The entire tools array, cached as a unit |
| Instructions | The system message |
| Conversation prefix | All input items except the final user message |

This ordering (tools → instructions → messages) matches Anthropic's required
cache order and works naturally with prefix-based caching on other providers.

### Gateway Behavior

For Anthropic, the gateway adds `cache_control: { type: "ephemeral" }` to
stable content, respecting the 4-breakpoint limit. Breakpoints are placed
after tools, after instructions, and after the conversation prefix.

For Gemini and OpenAI, no translation is required. Both providers cache
repeated prefixes automatically (OpenAI requires ≥1024 tokens).

### Usage Reporting

Gateways report cache performance via the existing `cached_tokens` field:

```typescript
response.usage.input_tokens_details?.cached_tokens
```

A non-zero value indicates the request benefited from caching. Clients should
not depend on specific caching behavior beyond this metric.

## Client Programming Model

Clients using `caching: "auto"` should structure requests to maximize cache
hits across all providers. The key constraints form a "union of limitations":

| Constraint | Implication |
|------------|-------------|
| Content-addressed | Byte-identical content required for cache hit |
| Prefix-based | Only leading content is cached; place stable content first |
| Ordered | Tools → instructions → messages (Anthropic requirement) |
| Minimum size | Content below ~1K tokens may not cache |

Following these constraints ensures good caching on all providers.

## Conformance

Gateways:

- MUST accept `caching: "auto"` without error
- SHOULD apply provider-appropriate caching when specified
- MUST report `cached_tokens` when caching occurs and provider reports it

Clients:

- SHOULD NOT depend on caching behavior beyond `cached_tokens` reporting
- SHOULD structure requests with stable content first

## When to Use

`caching: "auto"` benefits multi-turn conversations (the conversation prefix
grows but remains stable), applications with stable tools or instructions,
and any scenario where the same prefix appears in multiple requests.

For single requests with no reuse potential, the cache write overhead on some
providers (25% on Anthropic) may exceed benefits. A reasonable heuristic:
enable caching when `previous_response_id` is present.

## Cost Considerations

| Provider | Cache Write | Cache Read | Net Effect |
|----------|-------------|------------|------------|
| Anthropic | +25% | -90% | Significant savings after 2+ turns |
| Gemini | Free | Free | No cost impact |
| OpenAI | Free | -50% | Savings when cache hits |

## Future Work

Fine-grained cache control (TTL hints, content-level breakpoints, named caches)
is specified in [Proposal 002](./002-cache-control-hints.md).

## References

- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [Gemini Context Caching](https://ai.google.dev/gemini-api/docs/caching)
- [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)
