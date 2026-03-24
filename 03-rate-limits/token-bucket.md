# Token Bucket Algorithm — How Limits Actually Replenish

The rate limit numbers in the docs (RPM, ITPM, OTPM) are not reset-at-the-minute limits. They replenish continuously. Understanding this changes how you design agent traffic patterns — and explains why agent teams that fan out simultaneously hit limits their sustained usage rate doesn't predict.

---

## How the Token Bucket Works

Picture a bucket with a capacity equal to your rate limit. Every request drains the bucket. The bucket refills continuously at a rate of (limit / 60) per second — not all at once when the minute turns over.

**What this means in practice:**

| Misunderstanding | Reality |
|-----------------|---------|
| "60 RPM means 60 requests then wait for the minute to reset" | 60 RPM ≈ 1 request per second, replenishing continuously |
| "I can burst to my full limit at the start of the minute" | Bursting drains the bucket; you then wait for it to refill |
| "My average usage is within limits so I won't hit 429s" | Burst rate matters independently of average rate |

**The agent team fan-out problem:** Five agents starting simultaneously make 5 requests in under a second. At 60 RPM, that drains ~5 seconds of accumulated capacity instantly. Each subsequent request has to wait for the bucket to refill. The sustained rate is fine; the burst is the problem.

---

## The Three Limit Dimensions

| Limit | What it measures | Agent team exposure |
|-------|-----------------|---------------------|
| **RPM** | Number of API calls per minute | High — each agent makes independent calls |
| **ITPM** | Uncached input tokens per minute | Medium — prompt caching dramatically reduces this |
| **OTPM** | Output tokens generated per minute | High — parallel agents generate output simultaneously |

All three limits operate as independent token buckets. You can be within RPM and OTPM but exhausted on ITPM — or any other combination. Monitor all three.

---

## The ITPM Exemption Most Practitioners Don't Know

For all Claude models, only **uncached** input tokens count toward ITPM limits. Cached tokens are exempt.

In a well-designed agent system with stable system prompts and tool definitions:
- 60–70% of input tokens can typically be cached
- Effective ITPM headroom is **3–5x higher** than the nominal limit suggests
- Practitioners who don't know this are over-engineering rate limit handling and under-utilizing actual capacity

This is not a minor optimization. At 70% cache hit rate, you have effectively 3.3x your stated ITPM limit. See [cached-tokens.md](cached-tokens.md) for implementation.

---

## Acceleration Limits vs. Sustained Limits

These are separate mechanisms. You can trigger one without triggering the other.

**Sustained limits:** Your RPM/ITPM/OTPM measured over a rolling window. The standard rate limit.

**Acceleration limits:** The rate at which your traffic is *increasing*. Going from 0 to full concurrency instantly triggers this even if your sustained rate is well within limits.

**The canonical trigger:** Agent team launches all agents simultaneously. 0 → N concurrent requests in milliseconds. Acceleration limit fires.

**The fix:** Stagger agent startup. Add jitter. 1–2 seconds between agent launches is usually sufficient. See [monitoring-patterns.md](monitoring-patterns.md) for the staggered startup pattern.

---

## Operational Gotchas

**`count_tokens` calls consume RPM.** The token counting API endpoint counts against your requests-per-minute limit. If you're calling `count_tokens` before every request in a high-throughput agent loop, you're burning ~50% of your RPM budget on monitoring calls. Use selective counting (e.g., every 5th request, or only when context grows significantly) rather than every call.

**Different models have separate rate limit buckets.** Haiku 4.5 has separate limits from Sonnet 4.6 and Opus 4.6. A mixed-model agent team that exhausts Sonnet's ITPM doesn't affect the Haiku budget. Design accordingly — route high-volume worker traffic to Haiku, preserve Sonnet budget for orchestration and validation.

**Batch API has different rate limit handling.** Batch requests don't count toward standard RPM/ITPM/OTPM limits and process within 24 hours. For non-latency-sensitive workloads (overnight processing, bulk evaluation), Batch API is the rate limit escape hatch. Not covered in this section — see Anthropic's Batch API docs.

---

*Next: [Cached Tokens](cached-tokens.md) | [Monitoring Patterns](monitoring-patterns.md)*
