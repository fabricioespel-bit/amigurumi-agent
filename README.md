# Amigurumi Agent

A personal project with a dual purpose: a real product for my wife, who makes amigurumi (crochet dolls), and an exploration of **LangGraph** orchestration, **RAG** with a dedicated vector database, and **LLM observability** — gaps not covered by my other portfolio projects, which lean on Google ADK and managed RAG. Two modules: a **pattern generator** that produces a custom amigurumi recipe from a text description and/or a reference photo, and a **project assistant** that helps someone already crocheting — adapting patterns, converting techniques, calculating yarn, answering technique questions.

**Status: in progress.** Phases 0–3 of 6 are complete — domain research, the LangGraph generator module (Module A), a working React front-end consuming it end-to-end, and LLM observability via Langfuse. Vector search (Qdrant) and the RAG-based project assistant (Module B) are in progress.

## Architecture

### Module A — Pattern Generator (LangGraph)

```
intake_parser ──(low confidence)──→ clarify_with_user ──(answer)──→ intake_parser
      │
      └─(high confidence)─→ shape_planner → pattern_writer → validator ─┬─(invalid)──→ pattern_writer
                                                                          └─(valid)────→ formatter
```

- `intake_parser` — extracts a structured spec (subject, size, body family, customizations) from free text and/or a reference image, via a single multimodal call to Claude — no separate vision model.
- `clarify_with_user` — a real pause, not a retry: when the body-family decision is ambiguous, the graph uses LangGraph's `interrupt()` to stop and ask the user, resuming exactly where it left off once a `Command(resume=...)` arrives.
- `shape_planner` — deterministic, no LLM. Converts the spec into concrete stitch counts per round, sourced from a curated knowledge base derived from real crochet patterns — not invented at generation time.
- `pattern_writer` — writes the round-by-round recipe in Brazilian Portuguese crochet notation (pb, aum, dim...), one LLM call per body part.
- `validator` — recomputes the expected stitch count for every round and compares it against what the LLM wrote; on a mismatch, it routes back to `pattern_writer` with the specific error, up to a retry limit.

### Module B — Project Assistant (in progress)

RAG over a curated corpus of crochet techniques (text + reference diagrams, retrieved via hybrid dense+keyword search with reranking) in Qdrant, plus deterministic tools for yarn calculation and pattern scaling. Embeddings via Voyage AI (Anthropic's recommended pairing, since Claude has no native embedding endpoint), Qdrant self-hosted via Docker.

### Observability (Langfuse)

Every node in the LangGraph — including the deterministic ones, not just the LLM calls — is traced automatically: Langfuse's LangChain callback handler is injected once into the graph's invoke config, and LangGraph propagates it to every node without any per-node instrumentation. One non-obvious finding from wiring this up: a human-in-the-loop pause (`intake_parser` → `clarify_with_user` → resume) produces two separate LangGraph invocations, so by default it showed up as two disconnected traces. Fixed by passing the same LangGraph thread ID as `metadata.langfuse_session_id`, which groups both traces — pre-pause and post-resume — under one Langfuse session with aggregated cost.

## Key Design Decisions

**Deterministic validation, not LLM self-critique**
The stitch-count math the `validator` checks against is computed by plain Python from a hand-curated rules file, not asked of another LLM call. The `pattern_writer` is deliberately *not* given the target numbers up front — it has to compute them from a technique description, which is what makes the validator's check meaningful instead of a rubber stamp.

**Domain rules from real patterns, not invented**
The construction rules (stitch progressions, body-family branching, ear/hair technique variants) come from analyzing 11 real, purchased crochet patterns plus targeted public research — never reproduced verbatim, generalized into technique knowledge with provenance and confidence notes kept alongside the code.

**The end user shaped the design, not just the LLM**
An interview with the actual crocheter (a beginner, not an expert) changed concrete design decisions — most notably, that a recipe without a visual reference per technique isn't trusted by a beginner. That's why Module B's corpus pairs every technique with a diagram, and why the generator's output is structured explicitly by named body part instead of running text.

**Human-in-the-loop as graph state, not a UI hack**
When the body family (legs visible vs. a cone-shaped robe) can't be inferred confidently, the graph genuinely pauses mid-execution via LangGraph's `interrupt()` and checkpointer, rather than guessing or failing — tested end-to-end via LangGraph Studio and the real HTTP API.

## Project Structure

```
backend/
  app/
    main.py                    # FastAPI entrypoint — CORS, Langfuse callback wiring
    graphs/
      pattern_generator.py     # Module A — LangGraph StateGraph
      assistant.py             # Module B — in progress
    knowledge/
      construction_rules.py    # deterministic crochet construction rules
    rag/                       # in progress — Qdrant ingestion + retrieval
    tools/                     # planned — yarn_calculator, pattern_scaler
    models/                    # Pydantic schemas (spec, pattern)
  langgraph.json                # LangGraph Studio config
frontend/
  src/
    api/client.ts              # fetch calls to the backend
    pages/GeneratorPage.tsx    # form, human-in-the-loop UI, recipe display
    types.ts                   # TypeScript mirrors of the backend Pydantic models
docs/
  regras-construcao-amigurumi.md   # curated domain knowledge base
```

## Stack

- **LLM:** Claude (Anthropic), via `langchain-anthropic`
- **Orchestration:** LangGraph — `StateGraph`, conditional edges, `interrupt()` human-in-the-loop
- **Backend:** FastAPI · Python · uv
- **Observability:** Langfuse (Cloud) — per-node tracing, cost/latency, session-linked human-in-the-loop
- **Frontend:** Vite · React · TypeScript
- **Dev tooling:** LangGraph Studio (local graph debugging)
- **Vector DB (in progress):** Qdrant, self-hosted via Docker
- **Embeddings (in progress):** Voyage AI
- **Deployment (planned):** Cloud Run · Docker

## Related Projects

- [multi-agent-analytics](https://github.com/fabricioespel-bit/multi-agent-analytics) — a separate personal project exploring agent orchestration with Google ADK; this one deliberately explores LangGraph instead, to diversify the same underlying skill across frameworks
