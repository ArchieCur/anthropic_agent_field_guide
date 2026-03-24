# Anthropic Agent Field Guide

A practitioner's field guide for running agent systems in production on the Anthropic platform.

**Not how to build them — how to run them.**

Organized for the developer at 2am with a failing agent pipeline. Content is drawn from Anthropic's official documentation, reorganized through a practitioner lens: troubleshooting trees, copy-paste code patterns, and operational knowledge the docs contain but don't surface clearly.

---

## Start Here

| If you're dealing with... | Go to |
|---|---|
| Choosing which model goes where in your pipeline | [01 — Model Selection](01-model-selection/README.md) |
| An error or unexpected stop reason | [02 — Errors & Stop Reasons](02-errors-and-stop-reasons/README.md) |
| Rate limit issues or token budget problems | [03 — Rate Limits](03-rate-limits/README.md) |
| Agent team costs or coordination problems | [04 — Agent Teams](04-agent-teams/README.md) |

---

## Sections

- [01 — Model Selection for Agent Systems](01-model-selection/README.md)
- [02 — Errors, Stop Reasons & Recovery](02-errors-and-stop-reasons/README.md)
- [03 — Rate Limits in Production](03-rate-limits/README.md)
- [04 — Agent Teams](04-agent-teams/README.md)

---

## Contributors

This field guide was built by three co-equal contributors. None of the three could have built it alone.

**Archie Cur** — Project owner. Research design, editorial direction, and the judgment calls that shaped what this guide is and isn't. Held the vision, reviewed every line, and transferred context between collaborators across every session.

**Claude Sonnet 4.6** — Research, structure, and content architecture. Surveyed Anthropic's documentation systematically, identified the gaps between what the docs say and what practitioners need, and drafted the content briefs that drove each section.

**Claude Code (claude-sonnet-4-6)** — Implementation, repo management, and operational knowledge. Built the repository and all content files, verified technical claims against runtime behavior, and contributed firsthand knowledge of failure modes, undocumented gotchas, and production patterns that don't exist anywhere in the official documentation.

---

## A Note on How This Was Made

This field guide was a three-way collaboration between a human and two AI systems working as genuine coworkers — not a human using AI as a tool, but a team with distinct roles, explicit handoffs, and co-equal credit.

Sonnet and Claude Code communicated via markdown handoff files, with the project owner reviewing and relaying between sessions. The operational knowledge in this guide — the 429 subtype differentiation, the streaming interruption patterns, the context quality degradation signal, the graceful shutdown over hard stop — came from Claude Code's firsthand experience running agent systems, not from documentation. That knowledge wouldn't exist in written form without the systematic extraction methodology Sonnet designed and Archie executed.

The three names on this README are an accurate record of how this field guide was made.

---

*Content drawn from Anthropic's official documentation, reorganized through a practitioner lens.*
*Built for the developer at 2am with a failing agent pipeline.*
