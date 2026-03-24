# Section 03 — Rate Limits in Production

*Triage: if you're here, you're hitting 429s, burning through budget too fast, or trying to design before you hit the wall.*

---

## In This Section

- [Token Bucket Algorithm — How Limits Actually Replenish](token-bucket.md)
- [Cached Tokens and ITPM — The Advantage Most Practitioners Miss](cached-tokens.md)
- [Monitoring Patterns (copy-paste)](monitoring-patterns.md)

---

## Key Operational Facts

- Rate limits replenish **continuously** (token bucket), not at fixed intervals. You don't have to wait for the minute to reset.
- Cached input tokens **do not count toward ITPM**. In a well-structured agent system, this can give you 3–5x effective headroom.
- **Acceleration limits** are separate from sustained limits. Parallel agent fan-out can exhaust per-minute budget in seconds even if sustained rate is within limits.

---

