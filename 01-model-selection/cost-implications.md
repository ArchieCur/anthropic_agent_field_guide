# Cost Implications of Model Choice Across Agent Teams

Model choice is the largest cost lever in an agent system. This is not abstract — the math compounds fast, and the wrong assignment at the teammate level turns a viable system into an unviable one.

---

## The 7x Baseline

Agent teams use approximately **7x more tokens** than standard single-agent sessions. This is not a bug. It is the cost of parallel reasoning: each teammate runs its own full context window, accumulating its own conversation history, tool calls, and tool results independently.

Token usage scales roughly linearly with team size:

| Configuration | Approximate Token Multiple |
|---------------|--------------------------|
| Single agent | 1x |
| 2-agent team | ~2x |
| Full agent team | ~7x (Anthropic-documented baseline) |

The 7x figure is a floor for well-managed teams, not a ceiling.

---

## The Compounding Problem

Model rate multiplies on top of the token multiplier. The combination is where costs become unmanageable:

```
Standard session (Sonnet):    1x tokens  × Sonnet rate  = baseline
Agent team (Sonnet):         ~7x tokens  × Sonnet rate  = 7x baseline
Agent team (Opus):           ~7x tokens  × Opus rate    ≈ 21x baseline
Agent team (Haiku):          ~7x tokens  × Haiku rate   ≈ 2-3x baseline
```

Choosing Opus across all teammates does not buy you 7x a session — it buys you 21x a session. For the same reasoning quality at the teammate level, Sonnet is the correct choice.

---

## Tool Result Tokens: The Hidden Multiplier

This is underdocumented and will surprise you when you hit it.

Tool outputs — search results, file reads, API responses, database queries — are returned as part of the conversation and count against every agent's context window. Long tool outputs fed back into agent context accumulate across turns.

**The failure mode:** An agent executing 10 tool calls per task, each returning 2,000 tokens of results, adds 20,000 tokens per task to that agent's context — before any of its own reasoning. On a 5-agent team, that is 100,000 tokens per task cycle just in tool results.

**The fix:** Truncate or summarize tool results before returning them to the agent, not after. The right place to manage this is in the tool result handler, not in the agent's prompt.

```python
def truncate_tool_result(result: str, max_tokens: int = 1000) -> str:
    # Rough token estimate: ~4 chars per token
    max_chars = max_tokens * 4
    if len(result) > max_chars:
        return result[:max_chars] + "\n\n[Result truncated. Request specific section if needed.]"
    return result
```

---

## Cost Control Levers in Order of Impact

1. **Model selection per role** — largest lever by far. Get this right first.
2. **Tool result truncation** — prevents context window bloat from compounding across turns.
3. **Prompt caching** — cached input tokens are exempt from ITPM limits and reduce effective cost. Requires stable system prompts and tool definitions. See [Section 3 — Rate Limits](../03-rate-limits/README.md).
4. **Team size discipline** — each additional teammate is a full context window at full model rate. Do not add teammates speculatively.

---

## Real-World Reference Numbers

From Anthropic's documentation:

- Average Claude Code cost: **~$6/developer/day**
- Agent teams with Sonnet 4.6: **~$100–200/developer/month** (high variance — workload-dependent)

These figures assume Sonnet at the teammate level. Substituting Opus multiplies the agent team figure by approximately 3x. Substituting Haiku brings it down by approximately 60–70%.

---

*Next: [Decision Framework](decision-framework.md) | Back: [Role Assignment](role-assignment.md)*
