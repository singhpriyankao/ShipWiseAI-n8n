# ShipWise AI — Responsible AI Feature Decision Agent

A multi-agent system that helps a PM decide which model stack (GPT-4 / Claude / open-source) to use for a new customer-support AI feature — grounded in cost, performance, competitive, and regulatory research instead of days of manual spreadsheet/Slack digging.

A personal side project exploring AI product management end-to-end: PRD design, multi-agent architecture, and eval-driven development. Scoped to **customer support features in B2B SaaS** for MVP-1a.

## Architecture

One Orchestrator routes to three independently-built, independently-tested sub-agents:

```mermaid
flowchart TD
    U[PM asks a question via chat] --> O[Orchestrator<br/>GPT-4o + conversation memory]
    O -->|feature + cost/latency/accuracy targets| MSA[Model Stack Analysis<br/>benchmark table embedded in prompt]
    O -->|feature description| CA[Competitive Analysis<br/>competitor seed table embedded in prompt]
    O -->|feature + jurisdiction + data handling| REG[Regulations<br/>RAG over EU AI Act + GDPR]
    REG --> GATE{jurisdiction == EU?}
    GATE -->|no| FIXED[Deterministic fallback:<br/>"not covered" — no LLM call]
    GATE -->|yes| VEC[(Vector store retrieval)]
    MSA --> O
    CA --> O
    VEC --> O
    FIXED --> O
    O --> R[Synthesized, cited answer]
```

Built in [n8n](https://n8n.io); Orchestrator ↔ sub-agent calls use n8n's Execute Sub-workflow pattern (the Call n8n Workflow Tool node didn't work reliably in this build, so the routing was re-architected around a more stable connection type).

## Key design decisions

- **Grounding matched to corpus size, not defaulted to RAG.** Model Stack Analysis and Competitive Analysis embed small reference tables directly in the prompt; only Regulations — where the EU AI Act/GDPR corpus is genuinely too large for a prompt — uses real vector retrieval. Building RAG for a screen's worth of benchmark numbers would have solved a problem that didn't exist.
- **Model selection is a per-component tradeoff, not a default.** All three sub-agents use GPT-4o, but for three different stated reasons tied to the cost of being wrong in that spot — Regulations is the only "Highest priority" accuracy tier in the system, since a wrong answer there is a legal/reputational risk, not just an accuracy number.
- **A deterministic guardrail backs the highest-severity risk, not just a prompt.** Asking Regulations about a jurisdiction outside its ingested corpus (e.g. California/CCPA) didn't produce "I don't know" — vector similarity search always returns the *closest* chunk, and the model filled the gap with general knowledge despite being told not to. The fix wasn't a better prompt; it's a deterministic IF-node gate in front of the LLM call that returns a fixed "not covered" response for any non-EU jurisdiction, bypassing the model entirely.

## Eval framework

- **33 behavior-level test criteria** defined across 5 categories (Orchestrator, Competitive Analysis, Regulations, Model Stack Analysis, Feedback Collector), distilled into **16 automated regression test cases** (4 per built workflow).
- **Two metrics per Orchestrator test, not one**: an LLM-judge score grading the final answer against an explicit rule (not against similarity to an example output), plus a second, deterministic `correct_routing` metric — built after discovering that judge-only evals couldn't catch a real tool-misrouting/cross-contamination bug, since both bugs still produced plausible-sounding final text.
- All 4 workflows (Regulations, Competitive Analysis, Model Stack Analysis, Orchestrator) have working, debugged, automated offline eval pipelines — built end-to-end from dataset to judge to scoring, including working through real bugs along the way (malformed JSON bodies, silent Code-node failures, session-isolation issues).

**This is an offline regression suite, not production monitoring** — it re-runs a fixed golden set through the real workflow logic on every change. It has no connection to real user traffic yet; see Limitations below.

## Repo contents

| File | What it is |
|---|---|
| `Priyanka_PRD.docx` / `Priyanka_PRD_working.md` | The PRD — problem definition through pricing |
| `Evals/` | Behavior-level test criteria and eval schema spreadsheets backing the automated test suite |
| `Design/` | UI mockups for the eventual front end |

## Honest limitations (what this build does *not* prove)

1. **Static data, not live tools.** Benchmark and competitor tables are hand-embedded snapshots, no live pricing/search API.
2. **No production monitoring.** The eval suite can be 100% green and still miss real failure modes the fixed golden set never anticipated — nothing samples or grades real conversations yet.
3. **No learning loop.** Past recommendation outcomes never feed back into future ones (this is what the deferred Feedback Collector / Learning Agent, MVP-1b, is scoped to close).

Next step: rebuild on **Azure AI Foundry with a real front end**, specifically to close these three gaps — live data grounding, production-traffic evaluation, and an actual learning loop.

## Stack

n8n (workflow orchestration, AI Agent nodes, vector store) · OpenAI GPT-4o · Python / python-docx (PRD tooling)
