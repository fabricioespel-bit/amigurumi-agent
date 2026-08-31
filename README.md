# Amigurumi Agent

A personal project with a dual purpose: a real product and an exploration of **LangGraph** orchestration, **RAG** with a dedicated vector database, and **LLM observability** — gaps not covered by my other portfolio projects, which lean on Google ADK and managed RAG. Two modules: a **pattern generator** that produces a custom amigurumi recipe from a text description and/or a reference photo, and a **project assistant** that helps someone already crocheting — adapting patterns, converting techniques, calculating yarn, answering technique questions.

**Status: in progress.** Phases 0–5 of 6 are functionally complete — domain research, both LangGraph modules (pattern generator and project assistant, the latter with tool-calling and hybrid RAG), LLM observability via Langfuse, and a React front-end with a screen for each module. Phase 6 — validating with the actual end user (my wife, an amigurumi crocheter) — is underway and already changing the product: real usage surfaced a silent failure mode and a real ambiguity in how clothing was represented, both fixed below. Deployment is what's left.

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
- `shape_planner` — deterministic, no LLM. Converts the spec into concrete stitch counts per round, sourced from a curated knowledge base derived from real crochet patterns — not invented at generation time. Also decides which optional parts to add: a hat from a boolean field, and one separately named part per distinct clothing item the extraction step identifies (a vest and shorts become two labeled parts, not one generic "clothing" blob).
- `pattern_writer` — writes the round-by-round recipe in Brazilian Portuguese crochet notation (pb, aum, dim...), one LLM call per body part. For any part built with the standalone attachment technique, it's instructed to say so explicitly — crocheted separately and sewn on afterward, not integrated into the body piece — so the recipe states the assembly method instead of leaving it for the reader to guess.
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
The generator originally extracted customization requests (hat, clothing, hair) but never acted on them. Adding hat and clothing support was straightforward — both are round-by-round stitch progressions, the exact shape `PartPlan`/`validator` already checks. Hair wasn't, at first: it's a foundation chain plus individually knotted tufts, not an increasing round count, so forcing it into the same validated structure would have meant inventing a stitch-count model the source material doesn't describe — left out on purpose rather than papered over with a number that doesn't mean anything.

It came back once the real end-user validation flagged it as a gap worth closing before deployment, solved by making the mismatch explicit instead of hiding it: `PartPlan` gained a `validated` flag, and a part whose technique genuinely has no meaningful stitch count (hair's chain-plus-tufts) opts out of the geometry check rather than faking one. The trade-off is real and stated plainly rather than assumed away — that one part type is validated by construction (the code that builds it) instead of by re-derivation (the validator that checks it), which is a weaker guarantee than every other part in the recipe gets.

**A lossless format was the wrong default for a large generated image**
The illustration feature's first working version stored PNG data straight from the image API — about 2.5MB of base64 sitting in graph state from that point on, which every downstream node (and every Langfuse trace of them) would then carry. PNG's lossless compression doesn't help much on this kind of image; switching to JPEG with explicit compression cut the same image to roughly 120KB with no visible quality loss for a UI thumbnail. The fix wasn't a smaller resolution — the model's API rejects the smaller sizes outright — it was choosing the right encoding for what the file is actually for.

**A real user found a failure mode that no amount of testing the API surfaced**
The illustration call is deliberately silent on failure — cosmetic, shouldn't block a recipe — but silent turned out to mean invisible even to me: watching the raw JSON response during development, a missing image reads as "haven't tested this case yet," not as a failure. My wife's first real end-to-end use hit it as a recipe that simply had no picture, no error, nothing to click. Two fixes, neither of them about the image API itself: log the exception server-side instead of swallowing it, and have the front-end say "illustration unavailable this time" instead of rendering nothing — so a real failure and a feature that was never built look different to the person using it.

**Naming beats a boolean when the answer isn't yes/no**
A generated recipe for a character with both a vest and shorts produced one generic "Roupa" part covering both, with no line marking where one ended and the other began — because clothing was originally a single boolean flag, not a list of what was actually requested. Swapping `wants_clothing: bool` for `clothing_items: list[str]` let each named garment become its own labeled part, the same way the body already has separately named parts for head, body, and arms. The fix was recognizing that a "did the user want clothing" question was hiding a "which pieces of clothing" question underneath.

**Color as a qualitative zone, not a number the deterministic validator can check**
The same end-user validation surfaced two related gaps: recipes had no color information at all, and — the harder case — no marked point for switching colors mid-piece (e.g. a limb that's brown except for white paws at the tip). Colors are captured at intake as zones (`part`, `color`, `position`: whole piece / start / end), with a `"geral"` fallback for the default color of everything else. `pattern_writer` turns that into a per-part instruction, and when a part has more than one zone it explicitly tells the model to mark the exact round of the transition and work it in BLO (back-loop-only) for a clean line — reusing a technique already documented from the source patterns, not invented for this feature. The honest limitation: unlike stitch counts, the chosen transition round isn't deterministically checked — the `validator` verifies geometry, not color placement, so a mismatched round would currently pass silently. Registered as a known gap rather than quietly assumed correct.

**Exporting a PDF surfaced two library-behavior bugs no amount of reading the docs would have caught**
Adding a "download as PDF" button seemed like a UI-only feature, but the export endpoint reuses the pattern already sitting in the LangGraph checkpointer for that `thread_id` — `get_state(config).values["pattern"]` — instead of having the front-end resend the whole recipe, so no new state had to be introduced. Rendering it with `fpdf2` (pure Python, chosen specifically to avoid another native-dependency wall like the one that pushed both Qdrant and Langfuse to Cloud) hit two real bugs only found by generating actual PDFs and opening them: its core font can't encode an em dash or curly quotes — both of which the LLM writes routinely — so text has to pass through a small sanitization table first; and `multi_cell()` leaves the cursor at the right margin after each call, silently turning every subsequent full-width line into a zero-width one, which raises deep inside the library rather than failing at the call site that caused it.

**A second round of real-user feedback found a gap masquerading as a solved problem**
Safety eyes had been in the materials list since the very first version, but no recipe ever said which round to insert them in — a proportional placement formula was already researched and written down (head width ≈ 5x eye width, vertical position around round 9-12) but never wired into the prompt that writes the head. Freeform detail requests (a smiling mouth, a blush) had a similar gap one level up: `CustomizationFeature` was extracted but never reached `pattern_writer` at all. Fixing both meant adding a `part` field to `CustomizationFeature` (the same shape `ColorZone` already used) and, on first test, watching the model tag a facial detail as `part="rosto"` instead of `"cabeça"` — a plausible-but-wrong value that would have silently dropped the detail with no error. The fix was making the valid part names explicit in the intake prompt rather than hoping the model infers them, the same lesson the color and clothing features had already taught: an LLM extraction field with no closed vocabulary will eventually pick a synonym that matches nothing downstream.

**Faithful to the recipe, not just to the subject**
The illustration prompt only ever used `subject` and free-text `customizations` — never the recipe's own `colors`, `wants_hat`, `clothing_items`, or `wants_hair` fields, so the picture and the text could describe different dolls (wrong color, missing the requested hat). Rebuilding the prompt from every structured field the recipe already captures fixed that, and switching the style instruction from "illustrative crochet-doll style" to "realistic product photography" produced images far closer to what a crocheter expects as a reference. The honest limit stays: it's still a generative image, not a rendering computed from the actual stitch rounds, so it can suggest where a detail goes but can't guarantee it lines up with the round number the text gives — that guarantee comes from the text instructions, not the picture.

**Removing a promise the product never kept**
The form's placeholder text suggested a size in centimeters, but no field or logic anywhere ever used it — every recipe is generated at one fixed size regardless of what's typed. Once a second round of real usage flagged the mismatch, the fix wasn't to build size scaling; it was to stop implying a feature that didn't exist. Saying "one fixed size" honestly was better than a placeholder promising a knob that silently did nothing.

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
