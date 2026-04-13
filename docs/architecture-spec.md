# Libra — Technical Architecture Spec (v0/v1)

## 1. System Overview

Libra is an open-source, local-first knowledge base that uses LLMs to automatically compile, maintain, and evolve a structured wiki from ingested source documents. Unlike stateless RAG systems, Libra's wiki is a persistent, compounding artifact.

```
┌─────────────────────────────────────────────────────────┐
│                      Libra System                       │
│                                                         │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Ingest   │───▶│  Wiki Engine  │───▶│  Wiki Store   │  │
│  │  Pipeline │    │  (LLM Core)  │    │  (Git-backed) │  │
│  └──────────┘    └──────┬───────┘    └───────────────┘  │
│       ▲                 │                    │          │
│       │                 ▼                    ▼          │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  Source   │    │    Lint      │    │  Search Index  │  │
│  │  Store    │    │  Scheduler   │    │ (BM25+Vector) │  │
│  └──────────┘    └──────────────┘    └───────────────┘  │
│                                              │          │
│                                              ▼          │
│                                      ┌───────────────┐  │
│                                      │  Query Engine  │  │
│                                      └───────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 2. Three-Layer Data Model

Following Karpathy's architecture:

### Layer 1: Source Store (Immutable)
- **What:** Raw uploaded files — PDFs, articles, web pages, images, audio, plain text
- **Storage:** `sources/` directory, files stored with original name + SHA256 hash for dedup
- **Metadata:** `sources/manifest.json` — records each source's hash, ingest timestamp, original filename, extracted text path
- **Rule:** Sources are append-only. Never modified or deleted by the system.

### Layer 2: Wiki Store (LLM-Owned)
- **What:** Compiled markdown pages — entity pages, concept pages, topic summaries, cross-reference links
- **Storage:** `wiki/` directory, git-initialized. Each ingest = one commit.
- **Special files:**
  - `wiki/index.md` — Content-oriented catalog of all wiki pages, organized by topic
  - `wiki/log.md` — Chronological record of every ingest and update (what changed, which pages affected)
- **Cross-references:** Wiki pages link to each other via standard markdown links (`[Related Topic](./topic.md)`). These links form the dependency graph.
- **Versioning:** Git history provides full evolution tracking. Every page change is attributable to a specific source ingest.

### Layer 3: Schema (User-Configurable)
- **What:** A configuration file (`schema.md` or `schema.yaml`) defining:
  - Page types (entity, concept, summary, timeline, etc.)
  - Naming conventions
  - Required sections per page type
  - Domain-specific terminology and relationships
  - Custom ingest instructions (e.g., "always extract author bios," "track publication dates")
- **Co-evolved:** Users edit the schema; the LLM respects it during ingest/lint. Over time, the LLM can suggest schema improvements.

## 3. Core Operations

### 3.1 Ingest Pipeline

The most critical operation. Triggered when a user adds new source(s).

**Steps:**
1. **Parse** — Extract text/content from the source file (PDF parser, HTML-to-markdown, whisper for audio, OCR for images)
2. **Summarize** — LLM generates a structured summary of the source, following the schema
3. **Plan** — LLM reads `wiki/index.md` + relevant existing pages to determine:
   - Which existing pages need updates (affected pages)
   - What new pages should be created
   - What cross-references to add/update
4. **Execute** — LLM performs page creates/updates. Each page update is a targeted edit, not a full rewrite.
5. **Update index & log** — `index.md` updated with new pages; `log.md` gets an entry
6. **Commit** — All changes committed to git as a single atomic commit
7. **Re-index** — Search index updated with changed pages

**Cost optimization strategies:**
- **Batch coalescing:** If multiple sources are uploaded together, steps 1-2 run per-source but steps 3-6 run once for the batch (single planning pass, single commit)
- **Tiered models:** Use a fast/cheap model (e.g., Haiku) for routine cross-reference updates and index maintenance. Use a strong model (e.g., Opus/Sonnet) for synthesis of new content and complex page merges.
- **Dependency-scoped updates:** Only load pages that are directly linked to or topically related to the new source. The cross-reference graph is the dependency map — no need to scan all pages.

### 3.2 Query Engine

User asks a question against the compiled wiki.

**Steps:**
1. **Search** — Hybrid BM25 + vector search over wiki pages (NOT raw sources)
2. **Retrieve** — Top-k relevant wiki pages loaded as context
3. **Synthesize** — LLM answers the question using wiki pages, with citations back to specific pages
4. **Optional persist** — If the answer is valuable and novel, LLM creates a new wiki page from it (user-confirmable)

**Key distinction from RAG:** The search corpus is the compiled wiki, not raw sources. This means search hits are already synthesized, cross-referenced, and structured — yielding higher-quality retrieval.

### 3.3 Lint Scheduler

Periodic background maintenance of wiki quality.

**Checks:**
- **Contradictions** — Pages that make conflicting claims (flag for human review)
- **Stale claims** — Claims from old sources that may be outdated (based on source dates)
- **Orphan pages** — Pages not linked from any other page or the index
- **Missing cross-references** — Pages that discuss related topics but don't link to each other
- **Schema violations** — Pages that don't conform to the current schema structure
- **Gap analysis** — Topics mentioned frequently across pages but lacking a dedicated page

**Scheduling:** Configurable — could be on every N ingests, daily cron, or manual trigger. Results written to `wiki/lint-report.md` with actionable items.

## 4. Technology Stack (Recommended)

| Component | Recommendation | Rationale |
|---|---|---|
| **Language** | Python | Richest LLM/AI library ecosystem, fastest prototyping |
| **LLM Backend** | Pluggable (Anthropic, OpenAI, Ollama) | User choice; default to Claude for quality |
| **File parsing** | `unstructured`, `pymupdf`, `markdownify` | Broad format support |
| **Audio** | `faster-whisper` (local) | Privacy-preserving, no API cost |
| **Search** | `tantivy` (BM25) + `sentence-transformers` (vector) | Fast, local, no external dependencies |
| **Storage** | Filesystem + `gitpython` | Simple, versionable, portable |
| **Frontend** | Web UI (React or Svelte) | Browse wiki, upload sources, view evolution |
| **API** | FastAPI | Lightweight, async, good ecosystem |
| **Task scheduling** | `APScheduler` or `celery` (lightweight) | For lint and background jobs |

## 5. Directory Structure

```
libra-project/
├── sources/                  # Layer 1: immutable raw files
│   ├── manifest.json
│   ├── 2026-04-13_paper.pdf
│   └── 2026-04-14_article.html
├── wiki/                     # Layer 2: git-initialized, LLM-owned
│   ├── .git/
│   ├── index.md
│   ├── log.md
│   ├── lint-report.md
│   ├── entities/
│   │   ├── person-karpathy.md
│   │   └── org-openai.md
│   ├── concepts/
│   │   ├── retrieval-augmented-generation.md
│   │   └── knowledge-graphs.md
│   └── topics/
│       └── llm-knowledge-management.md
├── schema.md                 # Layer 3: user-configurable
├── config.yaml               # System config (LLM provider, model, schedule)
└── .libra/                   # Internal state
    ├── search-index/         # BM25 + vector index files
    └── cache/                # LLM response cache for cost control
```

## 6. API Surface (v1)

```
POST   /sources              # Upload new source(s), triggers ingest
GET    /sources              # List all sources with metadata
GET    /wiki                 # Browse wiki (index.md)
GET    /wiki/{page}          # Read a specific wiki page
POST   /query                # Ask a question against the wiki
POST   /lint                 # Trigger manual lint
GET    /lint/report           # View latest lint report
GET    /history              # Wiki evolution timeline (git log)
GET    /history/{page}       # Page-level change history
GET    /diff/{commit}        # View what changed in a specific ingest
GET    /schema               # Read current schema
PUT    /schema               # Update schema
```

## 7. Key Architecture Decisions & Tradeoffs

### Git-backed wiki vs. database
- **Chose:** Git-backed markdown on filesystem
- **Why:** Version history is free, diffs are free, evolution tracking is free. Users can inspect/edit wiki files directly. Portable — no database dependency.
- **Tradeoff:** Doesn't scale to millions of pages. Fine for v1 target (individual researchers, small teams).

### Search over wiki vs. search over sources
- **Chose:** Wiki pages are the primary search corpus
- **Why:** Wiki pages are already synthesized and cross-referenced. Searching compiled knowledge yields better results than searching raw text.
- **Tradeoff:** If wiki compilation has errors, search quality degrades. Lint helps catch this.

### Local-first vs. cloud
- **Chose:** Local-first with optional cloud sync
- **Why:** Privacy (research data can be sensitive), no ongoing infrastructure cost, works offline. Cloud sync can be added via git remote.
- **Tradeoff:** No real-time collaboration in v1. Acceptable for primary target (individual researchers).

### Batch coalescing vs. per-source ingest
- **Chose:** Batch coalescing when multiple sources uploaded together
- **Why:** A single planning pass over 5 sources is far cheaper than 5 separate passes, and produces more coherent cross-references.
- **Tradeoff:** Slightly more complex ingest pipeline. Worth it for cost savings.

## 8. Evolution Path

- **v0 (MVP):** CLI tool. Ingest files → generate wiki → query. No UI.
- **v1:** Web UI. Source upload, wiki browser, query interface, evolution timeline.
- **v2:** Multi-user / team support. Shared wikis, permissions, concurrent ingest.
- **v3:** Enterprise features (SSO, audit, compliance). Integrations (Slack, Confluence, etc.).
