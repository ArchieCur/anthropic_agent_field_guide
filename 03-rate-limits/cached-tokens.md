# Cached Tokens and ITPM — The Advantage Most Practitioners Miss

Prompt caching is the highest-leverage optimization available to agent systems. It reduces both cost and effective rate limit exposure simultaneously. Most practitioners don't use it because they don't know cached tokens are exempt from ITPM rate limits.

---

## What Gets Cached

| Content | Cache value | Notes |
|---------|-------------|-------|
| System prompts | Highest — identical across all calls | Must be static. Any change breaks the cache. |
| Tool definitions | High — shared across agent team members | Don't reorder or modify tools mid-session |
| Long static context | High — documents, reference material, shared instructions | Ideal for content all agents need |
| Conversation history prefixes | Medium — only the stable prefix is cached | Cache point moves forward as conversation grows |

---

## Cache Requirements

| Requirement | Value |
|------------|-------|
| Minimum cacheable block (Sonnet, Opus) | 2048 tokens |
| Minimum cacheable block (Haiku) | 1024 tokens |
| Cache TTL — standard | 5 minutes |
| Cache TTL — extended | 1 hour (now GA — no beta header required) |
| Cache hit confirmation field | `usage.cache_read_input_tokens` |
| Cache creation cost | 25% more than standard input tokens (first write) |
| Cache read cost | ~90% less than standard input tokens |

**On cache creation cost:** The first request that populates a cache entry costs 25% more than a standard uncached request. This is the write cost. Every subsequent request that hits the cache reads at ~10% of standard cost. Break-even is approximately 1.1 cache reads — after that, you're saving on every request.

---

## The ITPM Math

```
Standard agent call:      10,000 input tokens → 10,000 count toward ITPM
With 70% cache hit rate:  10,000 input tokens → 3,000 count toward ITPM
Effective ITPM capacity:  3.3x higher with caching vs. without
```

At 60% cache hit rate: 2.5x effective ITPM headroom.
At 70% cache hit rate: 3.3x effective ITPM headroom.
At 80% cache hit rate: 5x effective ITPM headroom.

---

## What Breaks Caching — Silent Performance Regressions

Cache invalidation is byte-level. Any change to a cached prefix — including invisible changes — breaks the cache for all subsequent requests.

| Cache breaker | How it happens | Fix |
|--------------|----------------|-----|
| Dynamic timestamps in system prompt | `f"Current time: {datetime.now()}"` | Remove from system prompt entirely |
| Session IDs or user names in system prompt | Personalizing the system prompt per user | Move to messages array, not system |
| Trailing whitespace changes | String concatenation edge cases | Strip and normalize before caching |
| Tool definition reordering | Dynamic tool list construction | Fix tool order, build list once |
| Adding/removing tools mid-session | Conditional tool availability | Include all tools, use instructions to constrain |
| Any tool definition modification | A/B testing tool descriptions | Stable tool defs per model deployment |

**The diagnostic signal:** If `usage.cache_creation_input_tokens` is non-zero on every request, the cache is being invalidated every call. You're paying the 25% write premium every time and never recovering the read savings. Check for dynamic content in the system prompt first.

---

## Implementation

### Basic: System Prompt and Tool Caching

```python
import anthropic

client = anthropic.Anthropic()

# System prompt with cache control — must be 2048+ tokens for Sonnet/Opus
system_prompt = [
    {
        "type": "text",
        "text": "Your full static system prompt here...",
        "cache_control": {"type": "ephemeral"}  # "ephemeral" = standard 5-min TTL
    }
]

# Cache the last tool definition — caches everything up to and including this point
tools_with_cache = list(tools)  # don't mutate the original
if tools_with_cache:
    last_tool = dict(tools_with_cache[-1])
    last_tool["cache_control"] = {"type": "ephemeral"}
    tools_with_cache[-1] = last_tool

response = client.messages.create(
    model="claude-sonnet-4-6",
    system=system_prompt,
    tools=tools_with_cache,
    messages=messages,
    max_tokens=4096
)

# Verify cache behavior
print(f"Cache hit:      {response.usage.cache_read_input_tokens} tokens")
print(f"Cache creation: {response.usage.cache_creation_input_tokens} tokens")
print(f"ITPM impact:    {response.usage.input_tokens} tokens (only this counts)")
```

### Automatic Caching (Minimal Setup)

Anthropic now supports automatic caching — the system caches the last cacheable block without manual `cache_control` placement.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    system="Your static system prompt...",
    messages=messages,
    max_tokens=4096,
    # No manual cache_control needed — automatic caching handles it
)
```

**Automatic Caching Notes:**  

- Automatic caching advances the cache point as conversations grow. Use manual `cache_control`   
when you need fine-grained control over exactly what is cached (e.g., caching both system prompt and tool definitions explicitly).
- Automatic caching is convenient but the cache point behavior is less predictable than manual placement. 
- If you upgrade from claude-sonnet-4-5 to claude-sonnet-4-6, the cache is invalidated.
For teams running long-running agent pipelines, a model version bump mid-session is a hidden cache killer.

---

## Agent Team Caching Strategy

**System prompts: make them static.** All dynamic content moves to the messages array.

```python
# Wrong — breaks cache on every call
system = f"You are a helpful assistant. Current date: {date.today()}. User: {username}."

# Right — static system prompt, dynamic content in messages
system = "You are a helpful assistant focused on [task]."
messages = [
    {"role": "user", "content": f"Context: today is {date.today()}, user is {username}. Task: ..."}
]
```

**Tool definitions: define once, share across agents.** All agents in a team using the same tool set means one cache entry serves the whole team.

**Shared context: cache it explicitly.** Reference documents or instructions all agents need go in a cached content block, not duplicated in each agent's system prompt.

**Per-agent variable content: messages array only.** Anything that varies per agent or per turn goes in messages — keeping the cacheable prefix stable across the entire agent team.

---

## Cache Hit Rate Targets

| System state | Expected cache hit rate |
|-------------|------------------------|
| New system, any dynamic content | < 20% — cache is not working |
| Stable system, first few sessions | 30–50% (cache warming) |
| Stable, optimized system | 60–80% |
| Mature system with long stable context | 80%+ |

**Realistic first-pass target: 50–60%.** The 60%+ figure assumes a fully optimized system with static system prompts, stable tool definitions, and no dynamic content leaking into the cached prefix. Start at 50% as your initial target and work up.

If cache hit rate is below 40% on a system with more than 2048 token system prompts, dynamic content is almost certainly the cause.

---

*Next: [Monitoring Patterns](monitoring-patterns.md) | Back: [Token Bucket](token-bucket.md)*
