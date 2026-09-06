# RAG in Production — Phase 1: The Data Backbone Nobody Demos

*Part 1 of a series on turning RAG proofs-of-concept into production-grade systems.*

## The demo that sold the project

Every RAG POC follows the same script. Someone picks twenty clean PDFs, a single embedding model, an off-the-shelf vector database, and wires them together with a retriever and an LLM call. It works. Stakeholders are impressed. A budget gets approved.

Then someone asks: *"Can we point this at our SharePoint, our Confluence, our claims database, and the 40,000 scanned contracts in the shared drive — and keep it updated as those change?"*

That single question is where most RAG projects quietly restart from zero. Not because the retrieval logic was wrong, but because the POC never had a **data layer** — it had a folder. Production RAG lives or dies on what happens *before* the vector database, not inside it.

This article is Phase 1 of a series documenting that POC-to-production journey. It covers the first five layers of the pipeline — ingestion through storage/indexing — with one goal: build a data preparation backbone that stays balanced across three forces that fight each other at every decision point.

## The triangle that governs every decision

Before touching a single component, put this on the whiteboard and keep it there for the rest of the project:

- **Scalability** — does this design still work at 100x the document count, 100x the source variety, and continuous updates instead of a one-time load?
- **Resilience** — what happens when a source is unavailable, a parser fails on a malformed file, or a downstream service throttles you? Does the pipeline degrade gracefully or fall over?
- **Cost** — what does this cost per document, per re-index, per month at production volume — not at POC volume?

Nearly every architectural choice below is really a trade-off between these three. A managed, pay-as-you-go extraction service is more resilient and scales without you managing infrastructure, but it costs more per document than a self-hosted OCR model that you now have to keep alive. There is no universally "correct" answer — there is only the answer that matches your document volume, your update frequency, and your budget. Treat every section below as a decision point on that triangle, not a recommendation to copy.

Security and governance are addressed as a dedicated topic at the end of this series, once the full pipeline is on the table — largely because if you build on cloud-native, enterprise-grade services throughout, a meaningful share of that burden (encryption at rest, access control propagation, audit logging) comes with the platform rather than something you build yourself.

---

## Layer 1: Data Ingestion — the layer that decides everything downstream

A POC ingestion step is a script that reads a folder once. A production ingestion layer has to answer three questions that the POC never asks: *what sources, what changed, and what triggers a re-run?*

**Source connectors as a contract, not a script.** Each source type — S3, SharePoint, Confluence, a relational DB, a web crawler, an internal API — should implement the same `fetch()` interface. The point isn't elegance; it's that adding source #12 next year shouldn't require touching sources #1 through #11. Whether each connector is a separate microservice or a component inside a shared ingestion service is itself a scalability/resilience/cost call: separate services isolate failure and scale independently, but they multiply operational overhead. A shared component is cheaper to run and simpler to deploy, but a bug in one connector's dependency can take down ingestion for every source. Choose based on how differently your sources actually behave and fail, not on a default preference for microservices.

**Change detection is really "what's the trigger?"** — and the answer is different for every source type:
- Object stores (S3, SharePoint document libraries) — event notifications (e.g., a file-added/updated/deleted event) that invoke a dedicated flow for that connector.
- APIs — an inbound request or scheduled poll invokes that connector's specific flow; you don't want one generic handler branching on source type at runtime.
- Legacy or relational databases — this is the case people underestimate. If there's no CDC feature available, the practical answer is often to extend the source application's own DML operations (insert/update/delete) to emit an event that triggers ingestion, rather than polling the whole table on a schedule. Where CDC or ETL/ELT tooling is available from your cloud provider or on-prem stack, use it — but confirm it's actually supported for your specific source before designing around it.

**Raw data store as insurance, not a nice-to-have.** Landing unprocessed source data before any parsing happens is what lets you replay a batch when a downstream parser gets a fix, or debug why a specific document came out malformed, without re-fetching from the source (which may have already changed or deleted the file). Skip this in a POC; don't skip it in production.

The output of this layer isn't "data" — it's a **handoff**: this ingestion pipeline should invoke the specific parsing/extraction flow appropriate to that connector and file type. That handoff is the seam where Layer 2 begins.

## Layer 2: Parsing & Extraction — usually the most expensive layer you didn't budget for

This is where POC assumptions break hardest. A POC picks documents that are easy to parse. Production hands you PDFs with embedded tables, scanned contracts, HTML with inconsistent markup, images, and audio transcripts — often all from the same source.

**No single parser survives contact with real documents.** You need a routing layer that identifies file type (and often sub-type) and dispatches to the right extraction path — sometimes OCR, sometimes a parser already validated in your POC, sometimes an enterprise-licensed extraction tool, sometimes a cloud-native pay-as-you-go service (Azure Content Understanding, Google Document AI, and similar). These are different cost profiles, different latency profiles, and different failure modes — which is exactly why they need to be interchangeable behind a routing decision, not hardcoded into the pipeline.

**Enterprise-grade extraction is usually worth the control it buys you.** A managed extraction service gives you better control over data quality, extraction consistency, and support — and that control tends to matter more than the per-document savings of a cheaper self-hosted alternative, provided the volume justifies the cost. This is frequently the most expensive line item in the entire pipeline, in both compute cost and engineering time. Budget it as such rather than discovering it during a production cost review.

**If you know your document universe, classify before you extract.** If you're building for a specific industry and know roughly what document categories you'll receive (say, 1,000 known document types), insert a classifier ahead of extraction. Once a document is classified, you know exactly what data you're looking for in it, which means you can move from generic extraction to schema-based extraction — pulling specific fields in a known shape instead of "everything, unstructured." This dramatically improves downstream data quality.

**Design for missing data as a first-class case, not an exception.** When a schema-based extraction comes back with a required field missing, that's ambiguous: either the extraction failed, or the information genuinely isn't in the source document. Build a human-in-the-loop checkpoint at exactly this point — not scattered everywhere, but at the specific decision where an automated system can't tell "extraction bug" from "data doesn't exist." Deciding where that checkpoint lives, and who reviews it, is an architecture decision, not an afterthought.

**Metadata extraction rides along, not behind.** Titles, authors, timestamps, source URLs, and — critically — permissions/ACLs need to be captured here, because ACLs captured at ingestion are what let you enforce access control at retrieval time later. If you don't carry permissions this far forward, you can't filter on them downstream without re-fetching from source.

## The decision point everyone wires an LLM into by default — and shouldn't

Look back at the last two layers for a second. The CDC/trigger router deciding which connector flow to invoke. The file-type router deciding which extraction path to use. The optional classifier deciding a document's category. The human-in-the-loop checkpoint deciding whether a missing field is an extraction bug or genuinely absent data. Every one of these looks like "a decision," and the instinct on most teams is to wire an LLM into each one. That instinct is usually wrong, and it's the biggest hidden line item in a production RAG budget that nobody puts on the architecture diagram — cost and, increasingly, carbon.

Two questions are worth asking before an LLM call goes anywhere near a pipeline decision point:

**Is this actually a decision, or a lookup?** If the outcome can be computed deterministically from data you already have — a file extension, a keyword match, a schema field, a fixed business rule, an ACL flag — write code, not a prompt. This is exactly what tool-calling (MCP tools are a good fit here) is for: give the model a deterministic tool that returns the field or the flag, instead of asking it to reason its way to an answer it could have looked up. That's not just cheaper and faster — a deterministic lookup can't hallucinate, where a model reasoning over raw context sometimes will. It also shrinks the *actual* judgment surface down to the residual set of genuinely ambiguous cases, which is the only place a model call was ever earning its cost.

**If it does need model judgment, does it need your biggest model?** Most of the decisions in this pipeline — is this document type A or B, does this response's citation match the source, should this go to OCR or the enterprise parser — are narrow, well-scoped classification calls. They're comfortably within a small or mid-tier model's reach. Save the frontier/reasoning-tier model for what actually needs it: multi-step planning, or the genuinely ambiguous edge cases a lighter model escalates rather than resolves on its own.

This is the idea behind **LLM routing**: a lightweight layer in front of your model calls whose only job is picking which model — or whether any model at all — handles a given task, instead of defaulting every task to one model end to end. Two patterns dominate in practice:

- **Classifier-based routing** — a small, cheap classifier looks at the incoming task before any generation happens and predicts which tier it needs, so simple tasks never reach the expensive model in the first place.
- **Cascade routing** — send the task to the cheapest model first, and only escalate to a stronger model if a confidence or quality check says the first answer wasn't good enough.

Neither is theoretical. RouteLLM (UC Berkeley, ICLR 2025) reported over 85% cost reduction on standard benchmarks while retaining roughly 95% of frontier-model quality, by sending only about 14% of queries to the strong model. FrugalGPT's cascade approach demonstrated cost reductions as high as 98% on some workloads. Production routing frameworks (LiteLLM and similar) now productionize this pattern — most teams no longer need to build a router from scratch, only decide where in the pipeline to put one.

The carbon angle is the same lever pulled from a different direction. Independent research comparing AI-generated and human-written code found emissions tracked more closely with the *number* of inference calls than with model size alone — and that imprecise or low-quality outputs from a cheap model triggering repeated retries can erase whatever savings the smaller model bought. Separately, energy-footprint studies of LLM inference put the inference phase — not training — as the majority driver of a deployed model's lifecycle emissions today, precisely because of call volume at scale, not per-call cost. Both findings point the same direction as the cost argument: the fewer unnecessary calls you make, and the smaller the model you make them with, the better you do on both axes at once. Treat carbon as a corollary of the cost corner of the triangle from earlier in this article, not a fourth force to juggle separately — the same discipline (call less, call smaller, call a deterministic tool first) improves both simultaneously.

Concretely, in the two layers above: the CDC/trigger router and the file-type router are almost always pure rule/metadata lookups — no model belongs there at all. The optional document classifier is a good candidate for a small classifier model rather than a frontier one. The human-in-the-loop checkpoint on missing fields is a good candidate for a lightweight confidence check that escalates to a person (not a bigger model) when it can't resolve the ambiguity itself. Keep this lens on every layer that follows in this series — query rewriting, re-ranking, and generation in Phase 2 all have the same pattern of "looks like it needs the biggest model, usually doesn't."

## Layer 3: Preprocessing & Chunking

Once extraction is producing clean, well-shaped data, chunking becomes comparatively low-risk — this is the one layer where established practice travels well. Clean/normalize (de-dup, whitespace, encoding), then isolate the chunking strategy itself fully behind a `chunk(document) -> [chunks]` interface, because this is the component you'll iterate on most: fixed-size gives way to semantic chunking, which gives way to hierarchical or sliding-window approaches as retrieval quality demands change. None of that should ripple into ingestion, extraction, or embedding — a clean interface here is what makes that iteration cheap. Tag every chunk with its parent document ID, position, and section headers; this is what makes citation and later filtering possible.

## Layer 4: Embedding

Similarly a lower-exercise layer: adopt a market-leading embedding model, and wrap it behind an adapter so that swapping models later — which happens more often than teams expect, as better or cheaper models release — doesn't touch anything else in the pipeline. The engineering effort here goes into the batch processor: batching, retries, and rate-limit handling need to be independent of which specific model is behind the adapter, so a model swap is a config change, not a rewrite.

## Layer 5: Storage & Indexing — the cost-vs-maintenance decision

The instinct from the POC is "one vector DB." Production forces a real choice: separate specialized stores (a vector DB, a document/content store, a relational metadata/filter store), or a modern unified platform that supports relational-style queries, semantic search, vector search, and hybrid (BM25 + k-NN) search in one system (Azure AI Search and comparable platforms are examples of this category).

A unified store is easier to maintain — one system instead of three, one place to reason about consistency. But it typically costs more per unit of data than assembling best-of-breed pieces yourself, and it introduces a dependency on that platform's roadmap. Multiple specialized stores can be cheaper at scale and give you sharper control over each piece, at the cost of more integration surface and more places for consistency to drift. This is squarely a cost-vs-maintenance exercise: run the infrastructure cost and the engineering-maintenance cost side by side for your actual volume and team size before deciding — not before choosing which vendor's demo looked best.

Whatever you choose, keep the write path narrow: the Vector DB Writer/Indexer should only know "take embeddings + metadata, upsert them." Nothing upstream should need to know which vector database is behind it, so that a Pinecone → Weaviate → pgvector migration (or a move to a unified platform) stays contained to this one component.

---

## The decision framework: three questions before you greenlight Phase 1

For every layer above, before signing off on a production design, ask:

1. **What breaks first when volume or source variety increases 10x?** Usually it's the parsing/extraction layer (cost and latency) or the ingestion triggers (a polling-based approach that quietly stops being real-time).
2. **What technical debt are you knowingly accepting, and is it worth it?** Skipping the raw data store to save infrastructure cost means no replay capability when a parser bug is found six weeks later — that's a real trade, not a shortcut.
3. **Which components are you willing to swap in 12 months, and did you actually isolate them?** If swapping your embedding model or vector database requires touching more than one layer, the interface boundaries above weren't real — they were aspirational.
4. **Does every model call in this pipeline actually need to be a model call — and does it need to be your biggest one?** Audit every decision point for a deterministic tool call or a smaller routed model before defaulting to your frontier LLM. This is the cheapest optimization in the entire architecture, and it pays off in both dollars and carbon.

## What's next

Phase 1 builds the data backbone: source to searchable index, balanced across scalability, resilience, and cost. Phase 2 of this series moves to the layers that sit on top of it — query processing, retrieval and re-ranking, context assembly, generation, orchestration, evaluation, and guardrails — where a different set of production failure modes shows up: retrieval that looks fine on a benchmark but degrades under real user queries, latency budgets that get eaten by re-ranking, and the observability you need to debug a bad answer three hops after it left the vector store.

The architecture diagram below maps every component and decision point discussed here into a single view of the data flow, from source connector to searchable index.
<iframe src="rag-architecture-phase1-diagram.html" width="100%" height="700" style="border:1px solid #ddd; border-radius:8px;" title="RAG in Production - Data Flow Architecture Diagram">
  <p><a href="rag-architecture-phase1-diagram.html">View the architecture diagram</a> (your browser does not support iframes).</p>
</iframe>

---

[← Back to all articles](../index.md) &nbsp;|&nbsp; [Phase 2: Query, Retrieval & Generation →](./rag-production-phase2-query-retrieval-generation.md)

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