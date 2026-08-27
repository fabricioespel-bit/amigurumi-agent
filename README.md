# Amigurumi Agent

A personal project with a dual purpose: a real product for my wife, who makes amigurumi (crochet dolls), and an exploration of **LangGraph** orchestration, **RAG** with a dedicated vector database, and **LLM observability** — gaps not covered by my other portfolio projects, which lean on Google ADK and managed RAG. Two modules: a **pattern generator** that produces a custom amigurumi recipe from a text description and/or a reference photo, and a **project assistant** that helps someone already crocheting — adapting patterns, converting techniques, calculating yarn, answering technique questions.

**Status: in progress.** Phases 0–5 of 6 are functionally complete — domain research, both LangGraph modules (pattern generator and project assistant, the latter with tool-calling and hybrid RAG), LLM observability via Langfuse, and a React front-end with a screen for each module, both exercised end-to-end in the browser. What's left: validating the product with its actual end user, and deployment.

## Architecture

### Module A — Pattern Generator (LangGraph)

```
intake_parser ──(low confidence)──→ clarify_with_user ──(answer)──→ intake_parser
      │
      └─(high confidence)─→ shape_planner → pattern_writer → validator ─┬─(invalid)──→ pattern_writer
                                                                          └─(valid)────→ formatter
                                                                                            │
                                                    end ←─(done)── refine_with_user ←───────┘
                                                                        │
                                                                  (wants changes)
                                                                        │
                                                                        ▼
                                                                  pattern_writer
```

- `intake_parser` — extracts a structured spec (subject, size, body family, customizations) from free text and/or a reference image, via a single multimodal call to Claude — no separate vision model.
- `clarify_with_user` — a real pause, not a retry: when the body-family decision is ambiguous, the graph uses LangGraph's `interrupt()` to stop and ask the user, resuming exactly where it left off once a `Command(resume=...)` arrives.
- `shape_planner` — deterministic, no LLM. Converts the spec into concrete stitch counts per round, sourced from a curated knowledge base derived from real crochet patterns — not invented at generation time. Also decides which optional parts to add (hat, simple clothing) from two explicit boolean fields the extraction step fills in, rather than fuzzy-matching the free-text customization list.
- `pattern_writer` — writes the round-by-round recipe in Brazilian Portuguese crochet notation (pb, aum, dim...), one LLM call per body part.
- `validator` — recomputes the expected stitch count for every round and compares it against what the LLM wrote; on a mismatch, it routes back to `pattern_writer` with the specific error, up to a retry limit.
- `refine_with_user` — the same `interrupt()` pattern as `clarify_with_user`, applied after the recipe is done: pauses to ask if the user wants any changes, loops back to `pattern_writer` with free-text feedback if so, ends otherwise. The front-end shows the finished recipe and the follow-up prompt at the same time, not one or the other.
- `formatter` — also generates a reference illustration of the amigurumi (OpenAI image generation), once per thread, cached in graph state so the follow-up editing loop doesn't regenerate it on every small text edit.

### Module B — Project Assistant

```
call_model ─(no tool call)──→ end
     ↑              │
     │        (tool call)
     └──── execute_tools
```

RAG over a curated corpus of crochet techniques (14 chunks, one per technique, manually split along the source document's own sections rather than by fixed size) in Qdrant Cloud, embedded via Voyage AI (Anthropic's recommended pairing, since Claude has no native embedding endpoint). Retrieval is genuinely hybrid: a dense vector search and a keyword match over chunk tags are merged and deduplicated, then reranked by Voyage's cross-encoder reranker — the vector search alone surfaces plausible-looking but not-quite-right chunks; reranking is what actually fixes the ordering.

The assistant itself is a small LangGraph — the classic tool-calling agent loop (`call_model` ↔ `execute_tools`) — with two deterministic tools: estimating yarn length from stitch count, and suggesting a yarn-thickness swap to resize a piece without recomputing stitches. Retrieval isn't a tool the model chooses to call; it runs unconditionally in code before the model ever sees the question, so grounding is guaranteed by construction rather than by hoping the model calls a retrieval tool. Reference diagrams per technique are planned but not yet sourced.

Not yet a StateGraph until tool-calling needed a real loop — the first version was a single retrieve-then-generate function, which was the right amount of complexity for what it did at the time. Exposed via `/api/assistant` and a chat-style front-end screen; each question is answered independently for now — the backend doesn't yet carry conversation memory across turns.

### Observability (Langfuse)

Every node in the LangGraph — including the deterministic ones, not just the LLM calls — is traced automatically: Langfuse's LangChain callback handler is injected once into the graph's invoke config, and LangGraph propagates it to every node without any per-node instrumentation. One non-obvious finding from wiring this up: a human-in-the-loop pause (`intake_parser` → `clarify_with_user` → resume) produces two separate LangGraph invocations, so by default it showed up as two disconnected traces. Fixed by passing the same LangGraph thread ID as `metadata.langfuse_session_id`, which groups both traces — pre-pause and post-resume — under one Langfuse session with aggregated cost.

## Key Design Decisions

**Deterministic validation, not LLM self-critique**
The stitch-count math the `validator` checks against is computed by plain Python from a hand-curated rules file, not asked of another LLM call. The `pattern_writer` is deliberately *not* given the target numbers up front — it has to compute them from a technique description, which is what makes the validator's check meaningful instead of a rubber stamp.

**Domain rules from real patterns, not invented**
The construction rules (stitch progressions, body-family branching, ear/hair technique variants) come from analyzing 11 real, purchased crochet patterns plus targeted public research — never reproduced verbatim, generalized into technique knowledge with provenance and confidence notes kept alongside the code.

**The end user shaped the design, not just the LLM**
An interview with the actual crocheter (a beginner, not an expert) changed concrete design decisions — most notably, that a recipe without a visual reference per technique isn't trusted by a beginner. That's why Module B's corpus schema carries an `image_url` per chunk (unfilled for now — sourcing real diagrams is separate work), and why the generator's output is structured explicitly by named body part instead of running text.

**Human-in-the-loop as graph state, not a UI hack**
When the body family (legs visible vs. a cone-shaped robe) can't be inferred confidently, the graph genuinely pauses mid-execution via LangGraph's `interrupt()` and checkpointer, rather than guessing or failing — tested end-to-end via LangGraph Studio and the real HTTP API.

**Cloud over self-hosted, decided by hitting the wall, not by planning ahead**
Both Langfuse and Qdrant were originally scoped as self-hosted (closer to the "operate the infra yourself" learning goal). Langfuse moved to Cloud because the self-hosted stack (Postgres+ClickHouse+Redis+MinIO) was disproportionate for a personal project. Qdrant moved to Cloud for a blunter reason: Docker Desktop no longer supports this machine's macOS version, and the lightweight alternative (Colima) failed to build its virtualization dependency on it too. Same conclusion reached two different ways — know when infra friction isn't the point of the exercise.

**A production bug caught by reading the observability tool, not by guessing**
A user report of a generic "failed to fetch" in the browser led nowhere by itself. Pulling the actual trace via Langfuse's API (not just its dashboard) showed exactly where it broke: Claude occasionally returns a structured-output field as a JSON-encoded string instead of a real list, which Pydantic rejects — an unhandled exception that took down the whole request. Fixed with a narrow `try/except` feeding the same retry path the validator already used, rather than a generic catch-all.

**A tool's signature has to be fillable from a sentence, not just reusable from code**
The yarn calculator's natural signature takes a full recipe object — every part, every round. That's the right shape when your own code calls it after generating a pattern, and the wrong shape for an LLM tool: the model can't infer a nested structure like that from "how much yarn do I need?" The tool-calling version takes a flat stitch count instead; the recipe-aware version stays for when a future feature hands the assistant an actual generated pattern to discuss.

**Proving RAG is grounding the answer, not just running**
Retrieval always executes — it's plain code, not something the model opts into — so "did it search?" is guaranteed by construction. The harder question is whether the model's answer actually uses what came back instead of its own training knowledge. Two forms of evidence, not just one: asking about a technique deliberately absent from the corpus (invisible decrease) and confirming the assistant admits it doesn't know rather than answering from memory; and pulling the exact trace from Langfuse to see the retrieved chunks sitting in the system prompt that produced a given answer.

**A feature that mechanically works but honestly can't do everything it looks like it does**
Reading a trace after testing the post-recipe editing loop turned up something worth stating plainly rather than glossing over: asking to change a structural choice (e.g. legs visible vs. a cone-shaped robe) doesn't actually change the construction technique — that decision was made once by `shape_planner` and the edit loop only revisits `pattern_writer`. The model complied by inserting a phrase into the instructions without changing the stitch math underneath, which reads as satisfied but isn't. Content-level edits within a part work correctly; edits that require re-deciding the body family don't yet, and that's tracked as a known gap rather than quietly accepted as done.

**Adding customizations selectively, because not all of them fit the same validation model**
The generator originally extracted customization requests (hat, clothing, hair) but never acted on them. Adding hat and clothing support was straightforward — both are round-by-round stitch progressions, the exact shape `PartPlan`/`validator` already checks. Hair isn't: it's a foundation chain plus individually knotted tufts, not an increasing round count, so forcing it into the same validated structure would mean inventing a stitch-count model the source material doesn't describe. It's left out on purpose, not by oversight — a mismatch between a feature request and the existing validation model is a reason to say so, not to paper over it with a number that doesn't mean anything.

**A lossless format was the wrong default for a large generated image**
The illustration feature's first working version stored PNG data straight from the image API — about 2.5MB of base64 sitting in graph state from that point on, which every downstream node (and every Langfuse trace of them) would then carry. PNG's lossless compression doesn't help much on this kind of image; switching to JPEG with explicit compression cut the same image to roughly 120KB with no visible quality loss for a UI thumbnail. The fix wasn't a smaller resolution — the model's API rejects the smaller sizes outright — it was choosing the right encoding for what the file is actually for.

## Project Structure

```
backend/
  app/
    main.py                    # FastAPI entrypoint — CORS, Langfuse callback wiring
    graphs/
      pattern_generator.py     # Module A — LangGraph StateGraph
      assistant.py             # Module B — LangGraph tool-calling loop
    knowledge/
      construction_rules.py    # deterministic crochet construction rules
    rag/
      corpus.py                # 14 curated technique chunks
      ingest.py                # embeddings (Voyage) + upsert to Qdrant
      retriever.py             # hybrid dense+keyword search with reranking
    tools/
      yarn_calculator.py       # stitch-count -> yarn length estimate
      pattern_scaler.py        # yarn-thickness swap for resizing
      illustrator.py           # reference image generation (OpenAI)
    models/                    # Pydantic schemas (spec, pattern)
  langgraph.json                # LangGraph Studio config
frontend/
  src/
    api/client.ts              # fetch calls to the backend
    pages/GeneratorPage.tsx    # form, human-in-the-loop UI, recipe display
    pages/AssistantPage.tsx    # chat-style UI for the project assistant
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
- **Vector DB:** Qdrant (Cloud) — hybrid dense+keyword retrieval with reranking
- **Embeddings/Reranking:** Voyage AI (`voyage-3`, `rerank-2`)
- **Image generation:** OpenAI (`gpt-image-1`) — reference illustration per recipe
- **Deployment (planned):** Cloud Run · Docker

## Related Projects

- [multi-agent-analytics](https://github.com/fabricioespel-bit/multi-agent-analytics) — a separate personal project exploring agent orchestration with Google ADK; this one deliberately explores LangGraph instead, to diversify the same underlying skill across frameworks
