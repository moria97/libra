# Libra — Product Specification (Draft v0.1)

*An open-source, LLM-powered knowledge base that compiles, evolves, and self-maintains.*

---

## 1. Vision

Libra is an open-source tool that transforms how people manage knowledge. Instead of querying raw documents every time (traditional RAG), Libra uses LLMs to **compile ingested sources into structured, interlinked wiki pages** that evolve and compound over time. Think of it as a personal research assistant that doesn't just answer questions — it builds and maintains a living knowledge base for you.

**One-liner:** "Your knowledge, compiled and compounding — not just retrieved."

**Inspiration:** Andrej Karpathy's [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

---

## 2. Problem Statement

Knowledge workers drown in documents — papers, articles, notes, meeting transcripts, reports. Current tools force a choice:

| Approach | Problem |
|---|---|
| **Manual wikis** (Notion, Confluence) | Maintenance burden kills adoption. Wikis decay. |
| **RAG tools** (NotebookLM, Quivr) | Stateless — every query re-derives answers. No compounding. |
| **AI note-taking** (Mem, Obsidian+AI) | Knowledge grows but isn't automatically synthesized or cross-referenced. |
| **Knowledge graphs** (Graphify) | Great for code structure, but one-shot — not continuous compilation. |

**The gap:** No existing product automatically distills ingested sources into evolving, compiled wiki pages that cross-reference, flag contradictions, and track knowledge growth over time.

---

## 3. Target Users

### Primary (v1): Individual Researchers & Knowledge Workers
- **Who:** PhD researchers, analysts, consultants, journalists, independent learners
- **Pain:** Managing 50-500+ sources across projects. Spending hours re-reading and manually synthesizing.
- **Job to be done:** "I want to drop in a paper/article/transcript and have my knowledge base automatically update — new entities added, existing pages enriched, contradictions flagged."
- **Willingness to pay:** Moderate. Comparable to Notion/Obsidian power users ($8-15/mo).

### Secondary (v2): Small Research Teams
- **Who:** R&D teams (3-10 people), competitive intelligence groups, consulting teams
- **Pain:** Shared knowledge is scattered across Slack, Google Docs, individual notes. No single source of truth that stays current.
- **Job to be done:** "Our team needs a shared knowledge base that automatically integrates everyone's research contributions."
- **Willingness to pay:** Higher ($20-50/user/mo).

### Tertiary (v3): Enterprise Knowledge Management
- **Who:** Large organizations with compliance, audit, and governance needs
- **Deferred:** Requires SSO, permissions, audit trails — only viable after PMF in v1/v2.

---

## 4. User Stories

### Core Loop (v0 — MVP)

**US-1: Ingest a source**
> As a researcher, I want to upload a PDF/article/markdown file so that Libra automatically generates or updates relevant wiki pages.

*Acceptance criteria:*
- User uploads one or more files via CLI or web UI
- System identifies key entities, concepts, and claims in the source
- New wiki pages are created for novel entities; existing pages are updated with new information
- Cross-references between pages are automatically maintained
- An entry is appended to the log with timestamp and summary of changes
- The index is updated to reflect new/modified pages

**US-2: Query the wiki**
> As a researcher, I want to ask questions and get answers synthesized from my compiled wiki, not re-derived from raw docs.

*Acceptance criteria:*
- User submits a natural-language query
- System searches compiled wiki pages (not raw sources) for relevant content
- Response includes citations to specific wiki pages
- Optionally, high-value query responses can be saved as new wiki pages

**US-3: Browse the wiki**
> As a researcher, I want to browse my knowledge base — see all pages, their relationships, and how they've changed over time.

*Acceptance criteria:*
- Index page shows all wiki pages organized by category
- Each page shows its content, cross-references, source citations, and last-updated timestamp
- Log shows chronological history of all changes

### Knowledge Quality (v1)

**US-4: Lint / health check**
> As a researcher, I want periodic automated checks that flag contradictions, stale claims, orphaned pages, and knowledge gaps.

*Acceptance criteria:*
- Lint can be run manually or on a schedule
- Report identifies: contradictions between pages, claims from outdated sources, pages with no cross-references, topic areas with sparse coverage
- User can review and accept/dismiss each finding

**US-5: Track knowledge evolution**
> As a researcher, I want to see how my knowledge base has grown and changed over time.

*Acceptance criteria:*
- Visual timeline of ingestion events and wiki changes
- Diff view showing how specific pages evolved
- Stats: total pages, sources ingested, entities tracked, contradictions resolved

**US-6: Customize the schema**
> As a researcher, I want to define how my wiki is organized — what categories exist, naming conventions, what metadata to track.

*Acceptance criteria:*
- Schema file (e.g., `schema.md`) defines wiki structure
- User can modify the schema; system reorganizes existing pages accordingly
- Different schemas for different domains (e.g., research papers vs. competitive intel)

### Collaboration (v2)

**US-7: Shared wiki**
> As a team lead, I want my team to contribute sources to a shared wiki that stays consistent.

**US-8: Source attribution**
> As a team member, I want to see who contributed which sources and when.

---

## 5. Competitive Positioning

### Positioning Statement
For researchers and knowledge workers who need to manage growing collections of information, Libra is an open-source knowledge compiler that automatically builds and maintains an evolving wiki from ingested sources. Unlike NotebookLM (session-scoped RAG), Notion AI (query layer over user-created content), or Mem (auto-organized notes), Libra **compiles knowledge into structured, cross-referenced pages that compound over time** — with automated contradiction detection and evolution tracking.

### Competitive Matrix

| Capability | Libra | NotebookLM | Mem.ai | Notion AI | Graphify |
|---|---|---|---|---|---|
| Auto-compiled wiki pages | **Yes** | No | No | No | Partial (--wiki flag) |
| Knowledge compounds over time | **Yes** | No | Partial | User-driven | No (one-shot) |
| Contradiction detection (lint) | **Yes** | No | No | No | No |
| Evolution tracking | **Yes** | No | No | Version history | No |
| Schema customization | **Yes** | No | No | Templates | No |
| Open-source | **Yes** | No | No | No | Yes |
| Local-first / data ownership | **Yes** | No | No | No | Yes |
| Multi-format ingest | **Yes** | Yes | Partial | Partial | Yes |
| Collaborative | v2 | Plus tier | No | Yes | No |

### Key Differentiators (in priority order)
1. **Compiled, not retrieved** — Wiki pages are pre-synthesized artifacts, not query-time RAG results
2. **Self-maintaining** — Lint operation catches contradictions, staleness, gaps automatically
3. **Evolution tracking** — Users see knowledge growth over time (unique in market)
4. **Open-source + local-first** — Full data ownership, no vendor lock-in
5. **Schema-driven** — Users control organization per domain

---

## 6. Feature Prioritization

### v0 — MVP (Prove the core loop)
| Feature | Priority | Notes |
|---|---|---|
| File ingest (PDF, MD, TXT) | P0 | Core loop — must work well |
| Auto wiki page generation | P0 | Entity pages, concept summaries |
| Cross-reference maintenance | P0 | Links between related pages |
| Index + log system | P0 | Navigation and audit trail |
| CLI interface | P0 | Fastest path to usable product |
| Query against compiled wiki | P1 | Search + synthesis from wiki pages |
| Git-backed storage | P1 | Version history for free |

### v1 — Knowledge Quality
| Feature | Priority | Notes |
|---|---|---|
| Lint operation | P1 | Contradiction detection, staleness, gaps |
| Evolution tracking / timeline | P1 | Key differentiator |
| Schema customization | P2 | User-defined wiki structure |
| Web UI for browsing | P2 | Visual wiki browsing experience |
| Additional ingest formats (web URLs, DOCX, audio) | P2 | Expand source support |
| Incremental ingest optimization | P1 | Only update affected pages |

### v2 — Collaboration & Scale
| Feature | Priority | Notes |
|---|---|---|
| Multi-user shared wikis | P1 | Team collaboration |
| Source attribution | P2 | Who contributed what |
| Pluggable LLM backend | P2 | Claude, GPT, local models |
| API for integrations | P2 | Connect to other tools |
| Export (Obsidian, Notion, HTML) | P3 | Portability |

---

## 7. Success Metrics

### v0 (MVP validation)
- **Ingest quality:** >80% of auto-generated wiki pages rated "useful" by users (manual review of first 50 test ingestions)
- **Cross-reference accuracy:** >90% of auto-generated links point to genuinely related pages
- **Ingest speed:** <60 seconds for a single document (under 20 pages)
- **GitHub traction:** 500+ stars within 3 months of launch

### v1 (Product-market fit)
- **Weekly active users:** 500+ (self-hosted + cloud)
- **Retention:** >40% weekly retention at 30 days
- **Lint value:** >50% of lint findings are actioned (not dismissed) by users
- **Knowledge base growth:** Average user ingests 10+ sources per month

### v2 (Growth)
- **Team adoption:** 50+ teams using shared wikis
- **Community contributions:** 20+ external PRs merged

---

## 8. Open Questions for Team Discussion

1. **Naming:** Is "Libra" the final project name? Any trademark concerns?
2. **First ingest format:** Should v0 support only markdown (simplest) or include PDF from day one?
3. **LLM provider for v0:** Default to Claude API? Or build with pluggable backend from the start?
4. **Hosting model:** CLI-only for v0, or include a minimal web UI?
5. **Monetization (if any):** Pure open-source? Or open-core with hosted/enterprise tier?
6. **Schema defaults:** Ship with pre-built schemas for common domains (research, competitive intel, learning)?
