# Role Assignment — Orchestrator, Worker, Validator

You are not picking a model. You are assigning a model to a role in a running system. Every role has different requirements. Conflating them is the most common and most expensive mistake in agent system design.

---

## The Three Roles

| Role | Responsibility | Key Requirement |
|------|---------------|-----------------|
| **Orchestrator** | Coordinates agent team, routes tasks, manages state | Reasoning quality, instruction fidelity |
| **Worker** | Executes specific task within defined scope | Speed, cost efficiency, tool use reliability |
| **Validator** | Checks outputs, enforces constraints, detects drift | Instruction following, consistency |

A single agent system may have one of each, multiples of the worker role, or no dedicated validator — but every agent in your system maps to one of these three. Assign accordingly.

---

## Model-to-Role Mapping

| Model | Best Role Fit | Why |
|-------|--------------|-----|
| `claude-opus-4-6` | Orchestrator on complex pipelines | Top-tier reasoning; handles ambiguous routing decisions where task boundaries are unclear |
| `claude-sonnet-4-6` | Orchestrator on standard pipelines; Validator | Frontier intelligence at substantially lower cost than Opus; more consistent on structured instruction-following |
| `claude-haiku-4-5-20251001` | Worker on well-scoped tasks | Fastest, cheapest, near-frontier performance on constrained tasks with clear inputs and outputs |

**Anthropic's explicit recommendation:** Sonnet for agent teammates — not Opus. This is not a cost-cutting suggestion. It reflects that Sonnet's instruction-following consistency at scale makes it the better fit for teammate roles.

---

## Key Practitioner Guidance

**Mixing models across roles is intentional design, not compromise.** A pipeline with an Opus orchestrator, Sonnet validators, and Haiku workers is not a half-measure — it is correct role assignment.

**The orchestrator does not need to be your most expensive model on every pipeline.** Sonnet is sufficient for orchestration when task routing is well-defined. Reserve Opus for pipelines where the orchestrator must reason through genuinely ambiguous or open-ended routing decisions.

**Validators need consistency more than raw intelligence.** Sonnet frequently outperforms Opus in validator roles because structured instruction-following — not open-ended reasoning — is what validation requires. An Opus validator is often over-engineered and more expensive for no gain.

---

## What Breaks When You Get It Wrong

**Opus on every worker agent:** Cost becomes unviable fast. The 7x token multiplier for agent teams × Opus rate ≈ 15x Haiku rate for equivalent throughput. See [cost-implications.md](cost-implications.md) for the math.

**Haiku as orchestrator on complex routing:** Reasoning gaps produce incorrect task distribution. Downstream agents receive malformed or underspecified instructions. The failure is often silent — Haiku will produce *an* answer, just not the right one. This surfaces as degraded output quality, not as an error.

**No dedicated validator role:** Output drift accumulates without detection. Drift in agent systems is positional — it compounds across turns and across agents — not dispositional. A system without validation has no mechanism to catch it early.

**Misidentifying worker task scope:** If the task handed to a Haiku worker is actually open-ended (requires judgment, handles edge cases, interprets ambiguous input), Haiku will attempt it and produce plausible-but-wrong output. The symptom is not an error — it is subtly incorrect results at scale. Match model capability to actual task complexity, not intended task complexity.

---

*Next: [Cost Implications](cost-implications.md) | [Decision Framework](decision-framework.md)*
