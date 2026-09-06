# RAG in Production — Phase 2: Query, Retrieval & Generation — Where Latency Meets Judgment

*Part 2 of a series on turning RAG proofs-of-concept into production-grade systems. [Phase 1](./index.md) built the data backbone — ingestion through storage. Phase 2 picks up the moment a query enters the system and follows it through to the response a user actually reads.*

## Same triangle, one new constraint

Phase 1's scalability / resilience / cost triangle still governs every decision below, but a fourth pressure shows up the moment a real user is waiting on the other end: **latency has a face now.** Nobody notices a two-second delay during nightly ingestion. Everybody notices it in a chat window. Every layer in this phase adds real, measurable milliseconds to a live request, so the math shifts from "what does this cost per document" to "what does this cost per request, inside this latency budget."

That shift also makes Phase 1's closing principle sharper, not softer: **audit every LLM call before it lands in a query-time layer.** A rewrite step, a HyDE generation, an LLM-based re-rank, a compression pass — each one is a real-time model call sitting directly in the user's critical path, not a background batch job you can amortize overnight. The routing discipline from Phase 1 — deterministic first, small model second, frontier model only when it's earned — applies with more force here, because an unnecessary call in this half of the pipeline isn't just a line on a monthly bill, it's latency someone is actively feeling.

---

## Layer 6: Query Processing — deciding what you're actually searching for

A POC sends the user's raw text straight to the vector database. Production can't, because real user queries are short, ambiguous, conversational, and often worded nothing like the source documents they're supposed to match.

### Query Rewriter / Expander

This is independent middleware, not a step bolted onto retrieval — which matters because it's the component you'll iterate on constantly as you discover new failure modes in production traffic. Three distinct techniques live here, and they are not interchangeable:

- **Query rewriting** — turns colloquial, fragmented input into clean search terms. *Example:* `"cheap flights nyc→sf next fri"` becomes `"Find flights from New York City to San Francisco departing Friday, [resolved date]."`
- **Query decomposition** — splits a multi-part question into sub-queries executed in parallel and merged afterward. *Example:* `"Compare GDP growth in India and Vietnam over the last decade and explain what drove it"` becomes four sub-queries: India GDP growth 2014–2024, Vietnam GDP growth 2014–2024, drivers of Indian growth, drivers of Vietnamese growth.
- **HyDE (Hypothetical Document Embeddings)** — asks an LLM to draft a fake, plausible-sounding answer first, embeds *that*, and searches with it. *Example:* for `"why do rising interest rates hurt bond prices?"`, the LLM generates a short explanatory paragraph about duration and discounted cash flows, and it's that paragraph's embedding — not the question's — that gets searched. This works because it turns a question-to-answer match into an answer-to-answer match, which is a much easier retrieval problem when your source documents are themselves written as answers (manuals, articles, explainers).

**The strong point:** don't run all three on every query. This is exactly the Phase 1 routing principle showing up again, one layer later. A cheap upfront classifier — keyword count, question-word presence, punctuation, conversational markers — can decide in microseconds whether a query needs rewriting at all. Route short, well-formed, fact-based queries straight through untouched. Route ambiguous or conversational ones to a lightweight rewrite model. Reserve decomposition for genuinely multi-part questions. Reserve HyDE for conceptual, abstract, or cross-lingual queries where vocabulary mismatch is the actual problem — not as a default step, because it is the most expensive and highest-risk of the three.

| | Traditional retrieval | Query rewriting | HyDE |
|---|---|---|---|
| **Mechanism** | Embeds the raw query directly | Modifies/expands/splits text before embedding | Generates a fake answer, embeds that instead |
| **Best for** | Direct, well-formed, fact-based queries | Ambiguous queries, multi-hop questions | Conceptual/abstract queries, low keyword overlap with source data |
| **Latency / cost** | Low | Medium — one fast, lightweight LLM call | High — a full generation step before retrieval even starts |
| **Risk** | Misses relevant context if phrasing diverges from source data | Low — generally improves or holds baseline quality | Hallucination risk — a wrong "fake answer" can send retrieval in the wrong direction entirely |

**Avoid HyDE specifically for:** factual lookups (exact numbers, IDs, names — there's nothing to "conceptually match," you need the literal fact), and for any system with a tight per-turn latency budget, since it requires two sequential LLM calls where a direct search needs zero.

### Does chat context belong in this layer?

**No — but this layer consumes it.** Conversation history is the responsibility of a separate **Context Manager / session store** (Redis, DynamoDB, Postgres — whatever holds session state), not the Query Processing layer itself. What the Query Processing layer does is take that history and fuse it with the new message into a single standalone query, through two specific operations:

- **Coreference resolution** — a lightweight LLM call resolves pronouns and implicit references. *Example:* after a user asks about "Product X," a follow-up of `"how much does it cost?"` is rewritten to `"What is the price of Product X?"` before it ever reaches the embedder.
- **Query condensation** — long, winding conversations get compressed into one to three core search terms so the vector database isn't confused by stale topics several turns back.

The boundary is clean and worth keeping clean: **storage and session lifecycle live outside this layer; context compression and transformation into a searchable query happen inside it.** Blurring that boundary is how teams end up with a query rewriter that also somehow owns Redis connection pooling.

### Query Embedder

This is not a new component — it's a strict reuse of the same embedding adapter built in Phase 1's Layer 4. **Vector consistency is non-negotiable here.** If ingestion embedded chunks with a fine-tuned model or a specific adapter layer on top of a base model (BGE, OpenAI, Cohere), the query must go through the identical weights and identical tokenization path. Even a minor mismatch — a different preprocessing step, a different model version — silently degrades every retrieval that follows, and it's one of the hardest production bugs to catch because nothing *errors*; results just get quietly worse. Treat "query embedder and ingestion embedder share one adapter, always" as an architectural invariant, not a convention.

---

## Layer 7: Retrieval — the layer where precision, latency, and access control collide

### Isolated retrieval strategies (vector, BM25, hybrid)

Running vector search, keyword/BM25 search, and a hybrid combiner as separable strategies — rather than one hardcoded path — is what lets you A/B them per query type instead of guessing which one your users need.

- **Vector search** wins on semantic/conceptual queries. *Example:* `"affordable eco-friendly laptops"` matches documents that say "budget sustainable notebooks" even with zero literal word overlap.
- **BM25/keyword search** wins on exact-match lookups where semantics actively hurt you. *Example:* a support query for `"error code E4032"` needs the literal token, not a semantically-nearby-but-wrong error code.
- **Hybrid** combines both via reciprocal rank fusion (RRF) or a learned re-weighting, and is the right default for mixed query populations — which is most production systems.

**The strong point:** isolating these as separate, swappable strategies buys you more than clean architecture — it buys you a fallback. If the vector database throttles or degrades, BM25 keeps the system answering something rather than failing outright. That's a resilience win, not just a tuning convenience. The cost is real, though: running multiple retrieval paths per query multiplies compute and adds the engineering burden of a merge layer (RRF or equivalent) that has to be built or hosted somewhere.

### Metadata Filter

This is where access control actually gets enforced, not just designed. It applies hard constraints — ACLs, date ranges, source restrictions, tenant IDs — before a document is allowed to surface, using exactly the permissions metadata Phase 1's extraction layer carried forward.

**Pre-filter vs. post-filter is a real trade-off, not a style choice:** filtering before vector search (restricting the search space to only documents the user can see) avoids ever surfacing a forbidden document, but on a sparse subset — a small tenant, a narrow date range — it can fragment the local vector space enough that recall quality drops. Filtering after retrieval keeps the vector search space dense and fast, but risks the top-N results getting mostly discarded by the filter, forcing a slow "page back and fetch more" loop. *Example:* a multi-tenant SaaS platform with thousands of tenants should almost always pre-filter by `tenant_id` (the security invariant is worth more than the recall cost), while a single-tenant system doing a coarse date-range filter can often afford to post-filter.

### Re-ranker

A cross-encoder or LLM re-scores the top-N candidates the initial retrieval pass returned, capturing token-to-token interactions between query and document that a vector similarity score physically cannot.

**The strong point, and it's the same one from Phase 1:** default to a cross-encoder, not an LLM, for re-ranking. Cross-encoders are purpose-built, fast relative to generative models, and swappable without re-indexing anything upstream. Reserve an LLM-based re-rank for the genuinely high-stakes fraction of traffic — legal, medical, or compliance-sensitive answers — where the extra hundreds of milliseconds and dollars are actually justified by the cost of being wrong. Also worth internalizing: **a re-ranker cannot rescue a bad first-pass retrieval.** If the initial vector/BM25 pass never captured the right document in its candidate set, no amount of re-ranking sophistication recovers it — which is exactly why Layer 7's retrieval quality, not just Layer 7's re-ranker, is where debugging effort belongs when answers are subtly wrong.

---

## Layer 8: Context Assembly — turning retrieved fragments into a prompt an LLM can actually use

### Context Builder / Compressor

Retrieval rarely hands back clean, non-overlapping, perfectly-sized context. This component is the editor sitting between "what we found" and "what we send."

- **Deduplication** — collapses near-identical chunks. *Example:* three SharePoint versions of the same policy document each surface a nearly identical paragraph; only one should reach the prompt.
- **Trimming / window fitting** — drops the lowest-scoring chunks to fit the model's context window with safety margin, because longer contexts cost more and add latency even when the model technically supports them.
- **Strategic ordering** — actively fights the "lost in the middle" effect, where LLMs recall content at the start and end of a prompt far better than the middle. *Example:* the single highest-relevance chunk gets placed first, the second-highest last, and lower-confidence chunks get buried in the middle rather than featured.
- **Compression / summarization** — when context is still too large, a linguistic compression method (LLMLingua and similar) or a small, fast model strips filler and condenses meaning. *Example:* a 2,000-token contract clause gets compressed to roughly 500 tokens while the specific obligations and dates survive intact.

**The strong point:** treat compression as another routing decision, not a default step. Deterministic/extractive compression (LLMLingua-style, or even simple sentence-scoring) should be the default because it's fast and cheap; escalate to a small LLM summarizer only when the compression ratio required is aggressive enough that extractive methods start losing meaning. Reaching for an LLM summarization call on every assembled context, by default, is the Layer 8 version of the mistake Phase 1 flagged in ingestion — a model call standing in for what should often be a deterministic transform.

### Prompt Template Manager

Hardcoded prompts inside application logic are a production liability, not a shortcut. This component version-controls templates (`v1.0_legal_advisor`, `v2.1_concise_summarizer`), injects standardized variables (`{{user_query}}`, `{{retrieved_context}}`, `{{chat_history}}`) right before generation, and decouples prompt wording from retrieval and API logic entirely — so changing a prompt's tone never requires touching the code around it. This is also where controlled experimentation lives: routing 10% of traffic to a new template variant to measure a real accuracy or satisfaction delta before a full rollout, the same discipline you'd apply to any other production change.

---

## Layer 9: Generation — where the pipeline's biggest cost decision lives

### LLM Adapter

The abstraction between application logic and whichever model API is behind it. Done properly, switching from one frontier model to another — or to a local model — is a configuration change: retrieval, prompt templates, and application code stay untouched. This adapter also owns resiliency mechanics that have nothing to do with which model is selected — retries, rate-limit backoffs, fallback models, and token-limit truncation — for the same reason Phase 1 kept the batch embedding processor independent of the embedding model behind it: swapping the model should never mean rewriting the plumbing around it.

### Output Parser / Validator

Raw LLM text is not a deliverable; it's an input to a validation step.

- **Structured output parsing** — coerces free text into a rigid schema (JSON, a Pydantic model) so downstream systems can consume it programmatically. *Example:* every response must conform to `{"answer": "...", "citations": [{"chunk_id": "...", "source": "..."}]}`.
- **Citation formatting and attribution** — maps generated text back to the specific chunks that were actually in context, and verifies the citation is real rather than invented. *Example:* if the model cites `chunk_id: 482` but chunk 482 was never in the assembled context for that request, the validator flags or strips it — this is the single most direct hallucination check available, because it's checking against known ground truth, not asking the model to grade itself.
- **Guardrails and hallucination checks** — validates the response doesn't contradict the retrieved context, contain banned content, or (for code generation) contain invalid syntax.
- **Graceful error repair** — when validation fails (malformed JSON, a missing field), the system either fixes it programmatically or triggers a self-correction loop: feeding the error back to the model with an instruction to correct only the structural problem, not regenerate the whole answer.

**The strong point, tying back to the whole series:** this is where the pipeline's largest single cost line sits, and it's exactly where Phase 1's cascade-routing pattern earns its keep in its purest form. Draft with a smaller, cheaper model by default; escalate to a frontier model only when the validator's confidence check fails, the query was flagged as high-stakes upstream, or the self-correction loop has already failed once. Defaulting every generation call to your most expensive model — the single most common production RAG mistake — is the Layer 9 instance of the exact anti-pattern this series has been flagging since Phase 1's ingestion layer: reaching for the biggest tool before checking whether a smaller one, or no model at all, was enough.

---

## The decision framework: four questions before you greenlight Phase 2

1. **What breaks first when concurrent request volume increases 10x?** Usually the re-ranker or HyDE step — both add fixed latency per request, and that latency doesn't parallelize away the way ingestion throughput does. Load-test these two specifically, not just the vector database.
2. **What technical debt are you knowingly accepting?** Skipping the re-ranker to hit a tight latency budget is a real, quantifiable precision trade — measure it, don't assume it away.
3. **Which components are you willing to swap in 12 months, and did you actually isolate them?** If replacing your re-ranker model or generation LLM touches your prompt templates or retrieval code, the adapter boundaries here weren't real.
4. **Does every model call in this half of the pipeline need to be a model call — and does it need to be your biggest one?** Query rewriting, HyDE, re-ranking, compression, and generation are five separate opportunities to route instead of default. Audit all five before signing off, because at query-time, every unnecessary call is latency a real person is waiting on.

## What's next

Phase 2 gets a query from the user's mouth to a validated response — query processing, retrieval, context assembly, and generation, all under a live latency budget. Phase 3 moves to the layer that wires all nine layers together and watches them: orchestration and caching, retrieval and generation evaluation, tracing that lets you debug a bad answer three hops after it left the vector store, and the input/output guardrails that catch what validation alone doesn't.

---

[← Phase 1: The Data Backbone Nobody Demos](./index.md) &nbsp;|&nbsp; [Back to all articles](../index.md)

## Comments

<script src="https://giscus.app/client.js"
        data-repo="uday-579/vajranex-knowledge-solutions"
        data-repo-id="R_kgDOUKZfgA"
        data-category="Announcements"
        data-category-id="DIC_kwDOUKZfgM4DEu_R"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
