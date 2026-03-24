# Monitoring Patterns

The 429 is the last signal, not the first. These patterns give you the first signal.

---

## Critical: Accessing Rate Limit Headers

**The header access gotcha that will silently break your monitoring:**

`client.messages.create()` returns a `Message` object. It does **not** expose HTTP response headers directly. Calling `response.headers` on a standard `Message` returns nothing useful.

To access rate limit headers, you must use the raw response:

```python
# Wrong — response.headers is not the HTTP headers
response = client.messages.create(...)
headers = response.headers  # AttributeError or empty

# Correct — use with_raw_response to get HTTP headers
raw = client.messages.with_raw_response.create(
    model=model,
    messages=messages,
    max_tokens=max_tokens
)
response = raw.parse()       # the Message object
headers = raw.headers        # the actual HTTP headers
```

All monitoring patterns in this section use `with_raw_response`. If you switch to standard `create()`, header monitoring silently returns `None` on every field with no error.

---

## Pattern 1 — Rate Limit Header Monitoring

Every API response includes rate limit headers. Read them. Most practitioners ignore them and learn about their limits from 429s instead.

```python
def log_rate_limit_status(headers) -> dict:
    """
    Extract rate limit headers from raw response.
    Call after every request in high-throughput agent loops.
    Alert threshold: 20% remaining on any dimension.
    """
    status = {
        "requests_limit":              headers.get("anthropic-ratelimit-requests-limit"),
        "requests_remaining":          headers.get("anthropic-ratelimit-requests-remaining"),
        "requests_reset":              headers.get("anthropic-ratelimit-requests-reset"),
        "input_tokens_limit":          headers.get("anthropic-ratelimit-input-tokens-limit"),
        "input_tokens_remaining":      headers.get("anthropic-ratelimit-input-tokens-remaining"),
        "input_tokens_reset":          headers.get("anthropic-ratelimit-input-tokens-reset"),
        "output_tokens_limit":         headers.get("anthropic-ratelimit-output-tokens-limit"),
        "output_tokens_remaining":     headers.get("anthropic-ratelimit-output-tokens-remaining"),
        "output_tokens_reset":         headers.get("anthropic-ratelimit-output-tokens-reset"),
        "retry_after":                 headers.get("retry-after"),
    }

    # Alert at 20% remaining on any dimension
    limit_pairs = [
        ("requests_remaining",     "requests_limit"),
        ("input_tokens_remaining", "input_tokens_limit"),
        ("output_tokens_remaining","output_tokens_limit"),
    ]

    for remaining_key, limit_key in limit_pairs:
        remaining = status.get(remaining_key)
        limit = status.get(limit_key)
        if remaining is not None and limit is not None:
            utilization = 1 - (int(remaining) / int(limit))
            if utilization >= 0.80:
                print(
                    f"WARNING: {remaining_key} — "
                    f"{remaining}/{limit} remaining ({utilization:.0%} consumed)"
                )

    return status


# Usage with with_raw_response
def call_with_monitoring(client, **kwargs):
    raw = client.messages.with_raw_response.create(**kwargs)
    response = raw.parse()
    status = log_rate_limit_status(raw.headers)
    return response, status
```

---

## Pattern 2 — Cache Performance Monitoring

```python
def log_cache_performance(response) -> float:
    """
    Monitor cache hit rate per request.
    Works on the standard Message object — no raw response needed.
    Target: 50-60%+ on stable agent systems.
    Low hit rate with large system prompts = dynamic content in the prefix.
    """
    usage = response.usage

    cache_hits     = getattr(usage, 'cache_read_input_tokens', 0) or 0
    cache_creation = getattr(usage, 'cache_creation_input_tokens', 0) or 0
    uncached       = getattr(usage, 'input_tokens', 0) or 0

    total_input = cache_hits + cache_creation + uncached
    hit_rate = cache_hits / total_input if total_input > 0 else 0.0

    print(
        f"Cache hit: {cache_hits:,} tokens ({hit_rate:.0%}) | "
        f"Cache creation: {cache_creation:,} | "
        f"Uncached (counts toward ITPM): {uncached:,}"
    )

    if total_input > 2048 and hit_rate < 0.40:
        print(
            "WARNING: Cache hit rate below 40% with large input. "
            "Check system prompt for dynamic content (timestamps, user IDs, session data)."
        )

    return hit_rate
```

---

## Pattern 3 — Staggered Agent Startup

Prevents the simultaneous burst that triggers acceleration limits. The fix is in how you launch agents, not in the retry handler.

```python
import asyncio
import random

async def launch_agent_team(
    agents: list,
    stagger_seconds: float = 2.0,
    jitter: float = 1.0
):
    """
    Launch agent team with staggered startup.
    Default: 2s between launches + up to 1s random jitter.
    For teams > 10 agents, increase stagger_seconds to 3-4s.
    """
    tasks = []
    for i, agent in enumerate(agents):
        delay = i * stagger_seconds + random.uniform(0, jitter)
        tasks.append(_launch_with_delay(agent, delay))

    return await asyncio.gather(*tasks, return_exceptions=True)


async def _launch_with_delay(agent, delay_seconds: float):
    await asyncio.sleep(delay_seconds)
    return await agent.run()
```

**Stagger guidance by team size:**

| Team size | Recommended stagger | Notes |
|-----------|--------------------|----|
| 2–3 agents | 1–2s | Minimal stagger needed |
| 4–7 agents | 2–3s | Standard team size |
| 8+ agents | 3–5s | Larger burst risk; wider stagger |

---

## Pattern 4 — Continuous Token Budget Tracker

Replaces the per-minute reset model. Token buckets replenish continuously — a per-minute reset is too coarse and creates false "safe" windows right after reset.

```python
import time
from collections import deque

class ContinuousTokenBudget:
    """
    Sliding window token budget tracker.
    Tracks actual token bucket replenishment (continuous) rather than
    per-minute resets (which don't match how the API actually works).
    """

    def __init__(self, itpm_limit: int, otpm_limit: int, rpm_limit: int,
                 window_seconds: int = 60, warning_threshold: float = 0.80):
        self.itpm_limit = itpm_limit
        self.otpm_limit = otpm_limit
        self.rpm_limit = rpm_limit
        self.window_seconds = window_seconds
        self.warning_threshold = warning_threshold

        # Sliding window: (timestamp, value) tuples
        self._input_events: deque = deque()
        self._output_events: deque = deque()
        self._request_events: deque = deque()

    def _prune(self, events: deque) -> None:
        """Remove events older than the window."""
        cutoff = time.monotonic() - self.window_seconds
        while events and events[0][0] < cutoff:
            events.popleft()

    def record(self, response) -> dict:
        """Record usage from a response and return current utilization."""
        now = time.monotonic()
        usage = response.usage

        # Only uncached input tokens count toward ITPM
        uncached_input = getattr(usage, 'input_tokens', 0) or 0
        output_tokens  = getattr(usage, 'output_tokens', 0) or 0

        self._input_events.append((now, uncached_input))
        self._output_events.append((now, output_tokens))
        self._request_events.append((now, 1))

        # Prune old events
        self._prune(self._input_events)
        self._prune(self._output_events)
        self._prune(self._request_events)

        # Current window totals
        itpm_used = sum(v for _, v in self._input_events)
        otpm_used = sum(v for _, v in self._output_events)
        rpm_used  = sum(v for _, v in self._request_events)

        utilization = {
            "itpm": itpm_used / self.itpm_limit,
            "otpm": otpm_used / self.otpm_limit,
            "rpm":  rpm_used  / self.rpm_limit,
        }

        for dim, ratio in utilization.items():
            if ratio >= self.warning_threshold:
                print(f"WARNING: {dim.upper()} at {ratio:.0%} capacity in sliding window")

        return utilization

    def headroom(self) -> dict:
        """Return remaining capacity across all dimensions."""
        self._prune(self._input_events)
        self._prune(self._output_events)
        self._prune(self._request_events)

        return {
            "itpm_remaining": self.itpm_limit - sum(v for _, v in self._input_events),
            "otpm_remaining": self.otpm_limit - sum(v for _, v in self._output_events),
            "rpm_remaining":  self.rpm_limit  - sum(v for _, v in self._request_events),
        }
```

**Why sliding window over per-minute reset:** A per-minute reset creates a false "full budget" signal at the top of every minute. The API sees your actual rolling window. If you sent 1000 requests at 11:59, you have limited budget at 12:00 — not a fresh full bucket. The sliding window matches the API's actual behavior.

---

## Priority Tier — When It's Actually Worth It

**Standard tier behavior:** Best-effort during overload. 529s increase during high-traffic periods. No SLA on availability.

**Priority Tier behavior:** 99.5% uptime target, prioritized compute allocation, predictable spend via usage commitments.

**When Priority Tier makes a material difference:**
- 529s are appearing in production logs with any frequency (even low frequency matters if the pipeline can't recover gracefully)
- Agent runs are long enough that a mid-run 529 + retry adds meaningful latency
- The pipeline is customer-facing or has downstream dependencies that amplify a single 529 into a visible failure

**When Priority Tier is not worth it:**
- You're seeing 529s only during known high-traffic windows and your retry handler absorbs them
- Your workload is batch/async and latency doesn't matter
- You haven't yet optimized caching and rate limit design — fix those first, they're free

**Confirmation in headers:** `anthropic-priority-input-tokens-limit` and `anthropic-priority-output-tokens-limit` headers confirm Priority Tier assignment is active.

---

## Monitoring Checklist

Before deploying an agent system to production:

- [ ] `with_raw_response` used wherever rate limit headers are read
- [ ] Rate limit headers logged at 80% consumption threshold (not just on 429)
- [ ] Cache hit rate tracked — target 50%+ at launch, 60%+ after optimization
- [ ] System prompt contains no dynamic content (timestamps, user IDs, session data)
- [ ] Agent team uses staggered startup, not simultaneous launch
- [ ] `count_tokens` calls not issued on every request in high-throughput loops
- [ ] Token budget tracked with sliding window, not per-minute reset

---

*Back: [Token Bucket](token-bucket.md) | [Cached Tokens](cached-tokens.md)*
