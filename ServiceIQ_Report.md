# ServiceIQ — One-Page Report

**Deliverable:** ServiceIQ RAG assistant answering field technicians' questions only from equipment documentation, built in four GenAI frameworks (LangChain, LlamaIndex, Semantic Kernel, Haystack) on OpenAI (`gpt-5.4-mini` for chat, `text-embedding-3-small` for embeddings).

## Architecture in one paragraph

ServiceIQ runs in two phases. **Indexing** (once per manual): `pypdf` extracts text → chunking (Haystack tab: 150-word chunks, 30-word overlap) → embeddings (`text-embedding-3-small`) → in-memory vector store. **Query** (per question): embed the question → retrieve top-k chunks → augment the prompt → generate with `gpt-5.4-mini` under a grounding rule: *answer only from the retrieved context; otherwise say "I don't know."* Tab 1 deliberately skips retrieval (LLM only) and Tab 3 sends truncated manual text, so the app demonstrates both the ungrounded and grounded behaviors side by side.

## Framework comparison (fill measured values from your runs)

| Criterion | LangChain | LlamaIndex | Semantic Kernel | Haystack |
|---|---|---|---|---|
| Lines of code for the core logic | *(fill)* | *(fill)* | *(fill)* | *(fill)* |
| Time to first working answer | *(fill)* | *(fill)* | *(fill)* | *(fill)* |
| Answered E-104 correctly? (yes/no) | *(fill — expected: yes, but ungrounded)* | *(fill — expected: yes, with sources)* | n/a (different task: planner) | *(fill — expected: yes, with passages)* |
| Refused the out-of-scope query? (yes/no) | *(fill — expected: no)* | *(fill — expected: yes)* | n/a | *(fill — expected: yes, "I don't know")* |
| Could you see the retrieved context? | No — no retrieval in this tab | Yes — `source_nodes` with scores shown | No — manual text truncated to 3,000 chars, no retrieval | Yes — top-3 passages displayed before generation |
| Ease of debugging (1–5) | *(fill)* | *(fill)* | *(fill)* | *(fill)* |
| One thing you liked | *(fill)* | *(fill)* | *(fill)* | *(fill)* |
| One limitation you hit | *(fill)* | *(fill)* | *(fill)* | *(fill)* |

## Reflection answers

**1. Why did Tab 1 answer the E-104 question even though it never saw the manual? Was the answer right?**
The prompt assigns a field-service role, and flow-meter faults after power interruptions are generic enough that the model produces confident, plausible steps from pretraining knowledge alone. It never saw the DX-500-specific sequence (supply-voltage check → J4 pulse-cable connector → controlled 60 s/90 s power cycle → calibration routine with a 20-litre certified canister → pulse sensor board DX-PM-114), so any match with the real procedure is coincidence. The deeper problem is that the answer is *unfalsifiable inside the app*: no source passages means no traceability, and an authoritative-looking but wrong repair step is a safety risk.

**2. What changed when you set split_overlap to 0 or top_k to 1?**
With `split_overlap=0`, any procedure crossing a 150-word chunk boundary is split; no single chunk contains the whole procedure, so retrieval returns a fragment and the generated answer silently drops steps. With `top_k=1`, only the single best-scoring chunk reaches the prompt, so a partially relevant top hit crowds out the chunk that holds the missing half — the answer under-covers or the model refuses. Overlap and top-k are insurance against boundary and ranking errors; the cost is duplicated tokens and more context per query.

**3. Which framework would you choose for ServiceIQ in production, and why?**
**Haystack** for the safety-critical answering core. Every stage is an explicit, individually testable component wired with `connect()` — splitter, embedders, retriever, prompt builder, generator — so retrieval can be inspected before generation, the grounding prompt is versioned and regression-testable, and components can be swapped (e.g., a hybrid retriever or a hosted document store) without rewriting the app. I would keep **Semantic Kernel** alongside it for the business workflow: its plugin model combines the semantic maintenance-planner function with native functions that write real work orders to a CMMS, which is exactly the enterprise-integration seam. **LlamaIndex** is the fastest way to stand up Q&A on a new manual during onboarding, and **LangChain** remains valuable for agents and query structuring — but for grounded answers, its lower step transparency is a liability in this domain.
