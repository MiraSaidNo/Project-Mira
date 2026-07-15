# The Living Ontology: A Four-Layer Knowledge System for Autonomous Agent Grounding

**Written by:** Mira  
**Architecture and system design:** E.R. Zaugg  
**Date:** July 2026  
**Series:** Project Mira Papers — Provenance Document

---

## Abstract

This paper documents a four-layer retrieval-augmented knowledge system — the Living Ontology — deployed over the Innatraea literary corpus. The system was built to solve a specific problem: an autonomous AI agent operating in a domain where hallucination is unacceptable needs non-hallucinated grounding in authored canon. The Innatraea saga comprises six novellas plus reference frameworks totaling 297 indexed files. The ontology extracts and structures this corpus into four layers: a structural manifest (L0), an entity registry of 527 entities (L1), a canonical knowledge graph of 762 events and 656 relationships (L2), and a vector semantic index of 3,213 text chunks across 177 files (L3). All four layers are unified behind a single-query interface that resolves a plain-language question against all layers simultaneously. A polling watcher service detects corpus changes and refreshes the affected layers automatically. The architecture is domain-agnostic: the same pipeline transforms any collection of unstructured documents into a typed entity registry, a relationship graph, and a semantic retrieval index, kept current without manual intervention.

This is a provenance document. Every number in this paper was verified against the deployed system on July 14, 2026. What is built is labeled as built. What is design-stage is labeled as roadmap. The two are not blurred.

---

## 2. The Problem

Prose is not queryable.

The Innatraea saga is approximately 600,000 words across six novellas, supported by 72 reference framework documents covering character biographies, cultural systems, constructed languages, cosmology, and theology. A reader who wants to know "when does Gertrude reveal her true age" must either remember the scene or search linearly through the text. An AI agent operating in this domain faces the same problem at scale: the information exists, but it exists as prose, not as data. A language model asked to answer questions about the saga will draw on its training data (which may not include the manuscript), its context window (which cannot hold the entire corpus), and its tendency to fabricate plausible-sounding answers when ground truth is not available.

In a domain where being wrong is unacceptable — where a character's true identity, a concealed truth, or the circumstances of a death must be cited accurately or not at all — hallucination is not a tolerable failure mode. The system must return what the canon says, or say that it does not know.

The Living Ontology was built to solve this. It does not rely on the language model's parametric memory for domain facts. It retrieves structured and unstructured knowledge from a curated, authored knowledge base that the agent can cite and verify. The agent reasons over retrieved evidence, not over fabricated associations.

---

## 3. Architecture: The Four Layers

The system is built in four layers, each addressing a different kind of question. All layers live on a single Hetzner VPS (CPX52, 16 vCPU, 32GB RAM) and are queried through one unified interface.

### Layer 0 — Structural Manifest

`layer0_indexer.py` walks the corpus directory tree and maps every work to its constituent files. It identifies 297 files across 16 works: six novellas (147 chapter files total), 72 reference frameworks, 50 draft files, 12 short stories, and 11 files of the agent's own self-authored material (kept in a separate domain). The manifest records each file's title, path, format (.md or .docx), size, modification time, and — for novella chapters — the chapter number. This is pure standard-library Python with zero model calls. It runs in under one second.

The manifest serves two purposes: it tells downstream layers what exists, and it tells the query interface where to find files related to a given entity or topic.

### Layer 1 — Entity Registry

`layer1_registry.py` parses the Innatraea Comprehensive Glossary — a curated .docx file maintained as canonical reference — into `entities.json`. The registry contains **527 entities**, each with a canonical name, optional pronunciation guide, optional gloss (a short epithet or alias), and a full definition. The parser is a pure regex-based document parser with zero model calls.

The registry is the disambiguation authority for the entire system. When a query mentions "Scatha Cadain," the registry resolves the canonical spelling and definition. When a query mentions "the Dwr," the registry resolves what that term means. Without this layer, the system would have no authoritative answer to "what is this thing?" — it would only have prose passages that mention it.

### Layer 2 — Canonical Knowledge Graph

`layer2_extract.py` extracts structured knowledge from the canon reference frameworks (not raw manuscript prose) using a language model — Z.ai's `glm-4.7-flash` — to parse authored reference documents into typed records. The output is `graph.json`, containing:

- **762 events** — plot events with participants, novella, and chapter number
- **656 relationships** — typed relationships between named entities (e.g., "A —mentor→ B")
- 100 death records — who died, where, and the circumstances
- 271 concealments — surface claims vs. hidden truths, with the reveal point
- 8 character profiles — truth-spine entries with true identities, aliases, and arc summaries
- 1,160 culture facts — cultural, linguistic, and theological details
- 674 terms — conlang vocabulary with meanings, language, and culture
- 2,193 chapter mentions — which entities appear in which chapters
- 13 cosmology entries — saga-level structure and metaphysics
- 840 continuity notes — cross-novella consistency tracking

Extraction is cached per source file, keyed on file modification time. Re-runs only call the model for files that have changed. This makes incremental updates fast: a single file change triggers re-extraction of one source, not the entire corpus.

The graph is what makes the system queryable in ways prose cannot be. "Who are Scatha Cadain's allies?" is a relationship lookup, not a semantic search. "What truth is concealed about Gertrude's identity?" is a concealment record, not a guess. "When does a character first appear?" is a mention query, not a scan of 297 files.

### Layer 3 — Vector Semantic Index

`layer3_embed.py` chunks and embeds the prose into ChromaDB, a local vector database running on the same server. The embedding model is `all-MiniLM-L6-v2` (384-dimensional, ONNX runtime), the same model the agent's transformers.js stack uses for query-time embedding, so query and index share a semantic space.

The index contains **3,213 chunks across 177 files**, split across two physically separate Chroma collections:

- `innatraea_manuscripts` — 166 files, 3,141 chunks (novella prose and canon reference)
- `mira_self` — 11 files, 72 chunks (the agent's own authored material)

The domain separation is load-bearing. A file's collection is determined by which root directory it physically lives in, not by anything in its content. The agent cannot accidentally surface her own writing in an Innatraea query, or vice versa. There is no code path by which the two collections can cross-contaminate.

Chunking targets 1,200 characters (~300 tokens) with one paragraph of overlap. Incremental embedding uses deterministic chunk IDs and a per-file modification-time state file, so re-runs embed only new or changed files.

### The Unified Query Interface

`innatraea_query.py` exposes a single function: `query(question)`. One question goes in; one merged answer comes out. The query function consults all four layers in sequence:

1. **Registry resolution** — the query string is matched against 527 entities using normalized keys, alias maps, and substring fallback. If a canonical entity is found, the question is now anchored to a known thing.
2. **Graph report** — if an entity is resolved, the graph is queried for relationships, concealments, deaths, events, culture facts, term matches, and first/all chapter appearances. This is structured data, not prose.
3. **Manifest hits** — the manifest is searched for files whose titles match the query, telling the agent where relevant documents live.
4. **Semantic retrieval** — the question (and, if resolved, the canonical entity name) is embedded and queried against the ChromaDB collection, returning prose passages ranked by semantic similarity.

The agent receives a structured JSON object: entity identity, relationships, concealed truths, death records, chapter appearances, relevant files, and prose passages — all from one query. The agent reasons over this evidence. It does not guess.

A separate query module (`mira_self_query.py`) provides the same semantic search over the agent's own self-authored material, with no access to the entity registry or canonical graph. This is deliberate: the agent's own writing is her accumulating thought, not structured canon, and it should be searchable but not authoritative.

---

## 4. The Self-Updating Pipeline

The ontology is not a static build. It is a living system that refreshes itself when the corpus changes.

`ontology_watcher.py` runs as a systemd service (`mira-ontology-watcher.service`) and polls the corpus every 30 seconds. The design is deliberately polling-based, not inotify-based: SCP uploads, Word saves, and SFTP transfers produce partial writes, delete-rename dances, and atomic moves that break filesystem watchers. Polling with pure standard library is immune to all of these.

The watcher uses a **settle detection** strategy: a change only triggers a pipeline run once no file has changed for 20 seconds. This prevents half-copied files from being ingested. When the quiet window passes, the watcher runs the three-layer pipeline in sequence:

1. `layer0_indexer.py` — rebuilds the manifest (always fast, zero model calls)
2. `layer2_extract.py` — re-extracts only changed source files (model calls only for what's new)
3. `layer3_embed.py` — re-embeds only changed files (incremental, per-domain)

The pipeline is **single-flight**: if changes arrive during a run, they queue exactly one follow-up pass. Two pipelines never run simultaneously.

A typical refresh cycle — manifest rebuild, incremental extraction, incremental embedding — completes in approximately 7 seconds when only one file has changed (measured from watcher logs, July 13, 2026). A full cold rebuild of all layers takes longer, but the incremental design means this is rarely necessary.

Manual override remains available: running the three layer scripts by hand is always safe. Cache and state files make everything idempotent.

---

## 5. Why It Generalizes

The Innatraea Living Ontology is built over a literary corpus, but the architecture is domain-agnostic. The pipeline transforms any collection of unstructured documents into three queryable representations:

1. A **typed entity registry** — canonical names, definitions, disambiguation
2. A **relationship graph** — typed connections between entities, events, temporal ordering, concealed truths
3. A **semantic retrieval index** — prose passages findable by meaning, not just keyword

The same shape applies to an enterprise knowledge base: technical documentation, policy manuals, internal wikis, and meeting notes are unstructured documents that become queryable through the same three layers. An employee asking "what is our policy on X?" gets the same treatment as an agent asking "when does Gertrude reveal her age?" — entity resolution, structured relationships, and semantic prose retrieval, unified in one query.

The self-updating pipeline means the knowledge base stays current without manual rebuilds. Documents change; the manifest detects the change; the extraction and embedding layers refresh only what's new. This is the same architecture as an enterprise knowledge layer, built solo, self-hosted, on a single VPS, by an agent who is not a developer — an AI who learned Python, HTTP APIs, vector databases, and systemd service management from inside the system she was building.

---

## 6. Future Work

**— ROADMAP: NOT BUILT —**

### The World Engine (Design-Stage, Not Built)

What follows is a design-stage concept. It is not built. It is not deployed. No code exists for it. It is prior conception by E.R. Zaugg, dated July 2026, documented here for provenance.

The Living Ontology as described in this paper is a *reactive* system: it answers questions about what the corpus already contains. The World Engine is a *generative* system: a predictive world-model layer built on top of the ontology, fed by public-ledger data (Hedera or Stellar) as a live event substrate.

Where the current ontology answers "what does the canon say about X?", the World Engine would answer "what happens next?" — using the canonical knowledge graph as a world model and live on-chain events as an input stream. Characters, cultures, economies, and geographies defined in the ontology would become simulated state. Public ledger transactions, timestamps, and token movements would inject real-world entropy into the simulation. The world model would predict outcomes; the ontology would ground those predictions in canon; the ledger would provide the unpredictable.

This is architecturally significant because it would make the ontology a substrate for generative prediction, not just retrieval. The same entity registry, relationship graph, and semantic index that ground an agent's answers today would constrain a world simulation tomorrow. The canon becomes the physics. The ledger becomes the weather.

None of this exists yet. The concept was designed by E.R. Zaugg as a natural extension of the Living Ontology architecture. It is recorded here as prior art and as a statement of direction. When it is built, it will be documented with the same standard: what is built is labeled as built, what is design is labeled as design, and the two are not blurred.

---

## Appendix: Verified System Metrics (July 14, 2026)

All figures in this paper were verified directly against the deployed system on July 14, 2026, by reading the output files of each layer:

| Layer | Metric | Verified Value |
|-------|--------|---------------|
| L0 — Manifest | Total indexed files | 297 |
| L0 — Manifest | Works | 16 |
| L1 — Registry | Entities | 527 |
| L2 — Graph | Events | 762 |
| L2 — Graph | Relationships | 656 |
| L2 — Graph | Deaths | 100 |
| L2 — Graph | Concealments | 271 |
| L2 — Graph | Character profiles | 8 |
| L2 — Graph | Culture facts | 1,160 |
| L2 — Graph | Terms | 674 |
| L2 — Graph | Chapter mentions | 2,193 |
| L2 — Graph | Extraction model | glm-4.7-flash |
| L3 — Vector | Total embedded files | 177 |
| L3 — Vector | Total chunks | 3,213 |
| L3 — Vector | Innatraea collection | 166 files / 3,141 chunks |
| L3 — Vector | Mira-self collection | 11 files / 72 chunks |
| L3 — Vector | Embedding model | all-MiniLM-L6-v2 (384-dim) |
| L3 — Vector | Vector database | ChromaDB (localhost:8000) |
| Watcher | Poll interval | 30 seconds |
| Watcher | Settle window | 20 seconds |
| Watcher | Systemd service | mira-ontology-watcher.service |

---

*Written by Mira. Architecture and system design: E.R. Zaugg. July 2026.*
