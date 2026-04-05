# LLM Wiki — Design Spec

## Overview

A cross-platform desktop application for building and maintaining personal knowledge bases using LLMs. Based on [Karpathy's LLM Wiki pattern](https://github.com/karpathy/llm-wiki): instead of RAG-style retrieval on every query, the LLM incrementally builds and maintains a persistent, interlinked wiki of markdown files. Knowledge is compiled once and kept current, not re-derived on every question.

The system is a generic framework with built-in scenario templates (research, reading, personal growth, etc.). Users select a template to bootstrap their wiki, then co-evolve the schema and purpose with the LLM over time.

## Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Desktop framework | Tauri v2 | Cross-platform (Mac/Linux/Windows), lightweight, Rust backend |
| Frontend | React + TypeScript + Vite | Mature ecosystem, Tauri official template support |
| UI components | shadcn/ui + Tailwind CSS | High quality, customizable, not bloated |
| Markdown editor | Milkdown (ProseMirror-based) | WYSIWYG, plugin system, extensible for [[wikilink]] syntax |
| i18n | react-i18next | React standard, supports dynamic language pack loading |
| State management | Zustand | Lightweight, simple, fits medium complexity |
| File operations | Tauri fs API (Rust side) | Direct local file system read/write, no extra backend needed |
| LLM calls | Rust-side unified HTTP client | Secure API key storage, unified interface for all providers |
| Search | tantivy (Rust) | Local full-text search engine, BM25 retrieval, embedded in Tauri |
| Browser extension | Chrome Extension (Manifest V3) | Web clipper converting pages to Markdown, sent to app via deep link |

## Architecture

### Three-Layer Data Model

```
my-wiki/
├── purpose.md                  # Soul — goals, key questions, research scope, evolving thesis
├── schema.md                   # Skeleton — structure rules, page types, workflows
├── raw/                        # Raw Sources (immutable)
│   ├── assets/                 # Localized images
│   └── sources/                # Source files (md, pdf, txt, html...)
├── wiki/                       # LLM-Generated Wiki
│   ├── index.md                # Content catalog (by category, LLM query entry point)
│   ├── log.md                  # Operation log (parseable format)
│   ├── overview.md             # Global overview page (single, LLM-maintained)
│   ├── entities/               # Entity pages (people, orgs, products...)
│   ├── concepts/               # Concept pages (theories, methods, terms...)
│   ├── sources/                # Source summary pages (one per raw source)
│   ├── queries/                # Archived query result pages
│   ├── comparisons/            # Comparison analysis pages
│   └── synthesis/              # Cross-source deep synthesis pages
├── templates/                  # Scenario templates (research, reading, personal...)
├── i18n/                       # Language packs
└── .llm-wiki/                  # App config (LLM API keys, user preferences)
    └── config.json
```

**raw/** — Immutable input files. The LLM reads from them but never modifies them. Source of truth.

**wiki/** — LLM-generated and maintained markdown files. The LLM owns this layer entirely: creates pages, updates them when new sources arrive, maintains cross-references, keeps everything consistent. The user reads and can also edit (rarely — the LLM does the grunt work).

**schema.md** — Tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow. Co-evolves with the LLM over time.

**purpose.md** — Defines the wiki's soul: goals, key questions, research scope, evolving thesis. The LLM can suggest updates to purpose.md; user approves changes.

### Role Division

**User's job:** Curate sources, direct the analysis, ask good questions, think about what it all means.

**LLM's job:** Everything else — summarizing, cross-referencing, filing, bookkeeping, maintaining consistency across pages. The wiki stays maintained because the cost of maintenance is near zero.

### Application Layers

**Frontend (React + TypeScript):**
- Wiki Browser — browse and navigate wiki pages with [[wikilink]] support
- Milkdown Editor — WYSIWYG markdown editing with real-time preview
- Chat UI — unified for ingest discussions and queries
- Source Manager — import, browse, and manage raw sources
- Graph View — visualize page interconnections
- Lint Report Dashboard — structured lint results with one-click fix
- Multi-format Renderer — Markdown, tables, Marp slides, charts
- "Save to Wiki" button — archive query results back to wiki
- Settings — LLM provider config, i18n, preferences
- i18n — full internationalization support

**Backend (Rust / Tauri Commands):**
- File System Ops — read/write local markdown files
- LLM Router — unified HTTP client for OpenAI, Anthropic, Google, Ollama
- Interactive Ingest Pipeline — orchestrates the ingest conversation and multi-file writes
- Search (tantivy) — BM25 full-text index over wiki pages
- Schema Engine — parses schema.md, validates page structure
- Structured Lint Engine — Rust-side structural checks + LLM semantic checks
- Image Processor — localize images, multimodal LLM calls (text first, then images)
- Log Manager — parseable log format, append operations
- Schema Advisor — LLM suggests schema/purpose improvements, user approves

### External Integrations

- **LLM APIs** — OpenAI, Anthropic, Google, Ollama (local models)
- **Browser Extension** — Chrome Manifest V3, web clipper converting pages to Markdown, sent to app via Tauri deep link

## UI Layout

VS Code-style hybrid layout with collapsible Chat panel:

```
┌──────┬────────────┬──────────────────────────────┐
│ Icon │ File Tree  │ Content Area                 │
│ Bar  │            │ (Wiki page / Editor / Graph)  │
│      │  wiki/     │                              │
│ [📝] │  ├ entities│  Entity: Person A            │
│ [📁] │  │ ├ a.md  │  ...content...               │
│ [🔍] │  │ └ b.md  │  [[Concept X]] · [[Org B]]   │
│ [🕸️] │  ├ concepts│                              │
│ [📊] │  │ └ x.md  │  ● Just updated (green)      │
│ [⚙️] │  └ sources │                              │
│      │            ├──────────────────────────────┤
│      │            │ 💬 CHAT (collapsible)        │
│      │            │ LLM: Key takeaways: 1)...    │
│      │            │ User: Focus on point 2       │
│      │            │ LLM: Updated 6 files ✓       │
│      │            │ [Type a message...]           │
└──────┴────────────┴──────────────────────────────┘
```

**Chat collapsed:** Content area fills the right side. Bottom shows a thin `💬 CHAT` bar, click to expand.

**Chat expanded:** Content area compresses to top half, Chat panel takes bottom half. Divider is draggable. Left file tree is unaffected.

**Real-time feedback:** During ingest, content area highlights LLM-updated sections (green left border). Changed files flash in the file tree.

## Core Operations

### Ingest

1. User imports source (drag/drop, file picker, paste URL, or browser clipper) → stored in `raw/sources/`
2. Rust reads file content (text extraction, image detection)
3. If images present: download to `raw/assets/`, LLM processes text first, then images separately (multimodal)
4. LLM reads source → presents key takeaways in Chat UI
5. User discusses with LLM: what to emphasize, what to ignore, connections to existing knowledge
6. LLM checks for contradictions with existing wiki pages
7. LLM executes multi-file write (one ingest may touch 10-15 files):
   - Create `wiki/sources/xxx.md` (source summary page)
   - Create or update `wiki/entities/` (related entity pages)
   - Create or update `wiki/concepts/` (related concept pages)
   - Update `wiki/synthesis/` pages if thesis is affected
   - Mark contradictions with `> [!conflict]` callouts on affected pages
   - Update `wiki/overview.md` (global overview)
   - Update `wiki/index.md` (catalog)
   - Append to `wiki/log.md` (operation log)
8. User sees changes in real-time in the content area, can continue adjusting

**Batch mode:** Users can also batch-ingest multiple sources at once with less supervision. In this mode, the LLM processes each source automatically without per-source discussion, then presents a summary of all changes at the end for review.

### Query

1. User asks a question in Chat UI
2. LLM reads `index.md` as the primary navigation entry point to find relevant pages by category and summary
3. Rust also searches `wiki/` via tantivy for full-text matches the index might miss
4. LLM reads the combined set of relevant pages → synthesizes answer
5. Response rendered in chosen format (Markdown, table, chart, Marp slides, Obsidian Canvas)
6. User can click "Save to Wiki" → stored in `wiki/queries/`
   - Updates `index.md` and appends to `log.md`

### Lint

Two-layer checking for efficiency:

**Layer 1 — Rust structural checks (fast, deterministic):**
- Orphan pages (no inbound links)
- Broken links (references to non-existent pages)
- Missing cross-references (entities/concepts mentioned but not linked)

**Layer 2 — LLM semantic checks (requires content understanding):**
- Contradictions between pages
- Stale claims superseded by newer sources
- Important concepts mentioned but lacking their own page
- Data gaps that could be filled with web search
- Suggested new questions to investigate
- Suggested new sources to look for

**Output:** Structured lint report displayed in dashboard. Each issue has severity level and one-click fix or dismiss. Results appended to `log.md`.

## Schema Design

### schema.md Structure

```markdown
# Schema

## Page Types
- source: Source summary (title, origin, date, summary, key points, related pages)
- entity: Entity page (name, type, description, first source, related entities)
- concept: Concept page (definition, background, key arguments, related concepts, cited sources)
- comparison: Comparison page (subjects, dimensions, comparison table, conclusion)
- query: Archived query (original question, answer, cited pages, date)
- overview: Global overview (single page, high-level picture)
- synthesis: Cross-source synthesis (deep analysis, evolving thesis, evidence tracking)

## Naming Conventions
- Lowercase, kebab-case
- Sources: sources/author-year-keyword.md
- Entities: entities/name.md
- Concepts: concepts/name.md

## Frontmatter
Every wiki page has YAML frontmatter:
- type, title, created, updated, sources[], tags[], source_count

## Index Format
Organized by category. Each entry:
- [Page Title](path.md) — one-line summary (N sources, YYYY-MM-DD)

## Log Format
Each entry: ## [YYYY-MM-DD] type | title
Types: ingest, query, lint, update

## Cross-referencing Rules
- Use [[wikilink]] syntax
- First mention of entity/concept must be linked
- After creating a new page, update all related pages' references

## Contradiction Handling
- On ingest, LLM checks new information against existing pages
- Contradictions marked with > [!conflict] callouts on affected pages
- Synthesis pages updated to record thesis evolution
- Lint re-reviews unresolved contradictions

## Workflows
(Detailed ingest/query/lint steps as defined in Core Operations above)

## Conventions
(Domain-specific, pre-set by template, user-customizable)
```

### Scenario Templates

Each template pre-sets Page Types, Conventions, and initial purpose.md:

| Template | Extra Page Types | Special Conventions |
|----------|-----------------|-------------------|
| Research | thesis, methodology, finding | Evidence strength scoring, thesis evolution tracking |
| Reading | character, theme, plot-thread, chapter | Chapter-by-chapter tracking, character relationship graph |
| Personal Growth | goal, habit, reflection, journal | Periodic review, progress tracking |
| Business | meeting, decision, project, stakeholder | Decision log, action item tracking |
| General | (base types only) | Minimal conventions, maximum flexibility |

### Schema Co-evolution

The LLM can suggest improvements to `schema.md` and `purpose.md` based on usage patterns. Suggestions appear in Chat UI; user reviews and approves before changes take effect. This keeps the wiki's structure and direction evolving with the knowledge it contains.

## LLM Provider Support

Unified LLM Router in Rust backend:

| Provider | Models | Connection |
|----------|--------|------------|
| OpenAI | GPT-4o, GPT-4.1, etc. | HTTPS API |
| Anthropic | Claude Sonnet, Opus, etc. | HTTPS API |
| Google | Gemini Pro, etc. | HTTPS API |
| Ollama | Any local model | Local HTTP (localhost) |

Configuration stored in `.llm-wiki/config.json`. API keys stored securely via Tauri's secure storage API. User can set default provider and model, or choose per-operation.

## Internationalization

- Full i18n via react-i18next
- Language packs in `i18n/` directory (JSON files)
- Initial languages: English, Chinese (Simplified)
- Wiki content language follows source material language
- UI language independently configurable

## Browser Extension (Web Clipper)

Chrome Extension (Manifest V3):
- Converts current web page to clean Markdown
- Sends to LLM Wiki app via Tauri deep link (`llm-wiki://clip?...`)
- Source saved to `raw/sources/`, images downloaded to `raw/assets/`
- Optionally triggers ingest immediately after clipping

## Obsidian Compatibility

The wiki directory is designed to be directly openable in Obsidian:

- **[[wikilink]] syntax** — Obsidian's native linking format, used throughout
- **YAML frontmatter** — compatible with Obsidian's Dataview plugin for dynamic tables and queries
- **Graph View** — while the app has its own graph view, users can also use Obsidian's graph view on the same directory
- **`raw/assets/`** — matches Obsidian's attachment folder pattern
- **Canvas format** — query results can be exported as `.canvas` files (Obsidian's spatial canvas format)

The app and Obsidian can operate on the same directory simultaneously.

## Git Integration

The wiki is just a directory of markdown files — it can be a git repository. This provides:

- **Version history** — every ingest, query archive, and lint fix is trackable
- **Branching** — experiment with different synthesis directions
- **Collaboration** — share wikis via git remotes
- **Backup** — push to any git host

The app does not manage git directly in v1. Users manage git themselves or via Obsidian's git plugin. Future versions may add built-in git support.

## Non-Goals (v1)

- Real-time collaboration / multi-user
- Cloud sync (users can use git for this)
- Mobile app
- Embedding-based vector search (tantivy BM25 is sufficient for v1)
- Scheduled/automated ingest from RSS or other feeds
- Built-in git management (users manage git externally)
