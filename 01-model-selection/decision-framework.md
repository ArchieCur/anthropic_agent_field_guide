# Model Selection Decision Framework

*Copy-paste reference. Answer three questions, get your model.*

---

## The Three Questions

1. What is this agent's primary responsibility? (orchestrate / execute / validate)
2. How constrained is the task? (open-ended reasoning vs. structured scope)
3. Is this agent on the critical latency path?

---

## Decision Tree

```
Is this agent coordinating other agents?
├── YES → Is the pipeline complex with ambiguous routing decisions?
│         ├── YES → claude-opus-4-6
│         └── NO  → claude-sonnet-4-6
└── NO  → Is this agent validating or enforcing constraints?
          ├── YES → claude-sonnet-4-6
          └── NO  → Is the task well-scoped with clear inputs/outputs?
                    ├── NO  → claude-sonnet-4-6
                    └── YES → Is this agent on the critical latency path?
                              ├── YES → claude-haiku-4-5-20251001  (fastest)
                              └── NO  → claude-haiku-4-5-20251001  (cheapest)
```

**Note on the latency branch:** Haiku is the right call for well-scoped workers regardless of latency requirements. The branch is included to reinforce that Haiku's speed advantage is a bonus, not a tradeoff — you are not sacrificing quality on constrained tasks.

---

## Copy-Paste Model Strings

```python
# Orchestrator — complex pipeline (ambiguous routing, multi-step reasoning)
ORCHESTRATOR_COMPLEX = "claude-opus-4-6"

# Orchestrator — standard pipeline / Validator
ORCHESTRATOR_STANDARD = "claude-sonnet-4-6"
VALIDATOR = "claude-sonnet-4-6"

# Worker — well-scoped tasks with clear inputs/outputs
WORKER = "claude-haiku-4-5-20251001"
```

---

## Migration Gotchas

If you are upgrading from Claude 3.x to Claude 4, these will break your system. They are not warnings — they are hard failures.

| Gotcha | Behavior | Fix |
|--------|----------|-----|
| `temperature` + `top_p` together | 400 error on Claude 4 models | Use one or the other, not both |
| Legacy tool versions | Unsupported, will error | Update to `text_editor_20250728`, `code_execution_20250825` |
| `undo_edit` command | Removed entirely | Remove from any tool call logic |
| Prefilling assistant messages on Opus 4.6 | Returns 400 error | Opus 4.6 does not support prefill — remove or route to Sonnet |
| Haiku 4.5 rate limits | Separate limits from Haiku 3.5 | Re-check rate limit tier after migration — do not assume carry-over |

**On the prefill gotcha:** This one is easy to miss because prefill works on Sonnet and Haiku 4.x. If you have a mixed-model pipeline and only test the orchestrator with Sonnet, the Opus path will pass local testing and fail in production. Test each model role explicitly.

---

## Quick Sanity Check Before Deploying

- [ ] Every agent in the system maps to exactly one role (orchestrator / worker / validator)
- [ ] No worker agents are assigned Opus without explicit justification
- [ ] Tool result handlers truncate or summarize before returning to agent
- [ ] System prompt is static (dynamic content breaks prompt caching)
- [ ] If using Claude 4 models, `temperature` and `top_p` are not both set
- [ ] If using Opus 4.6, prefill is not used anywhere in that agent's call path

---

*Back: [Role Assignment](role-assignment.md) | [Cost Implications](cost-implications.md)*
