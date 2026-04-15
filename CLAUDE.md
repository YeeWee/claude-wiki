# CLAUDE.md — LLM Wiki Schema

> This file is the schema for a persistent, LLM-maintained personal knowledge base (wiki).
> It tells Claude how the wiki is structured, what conventions to follow, and how to behave
> during ingestion, querying, and maintenance. Claude owns the wiki layer; the human owns sourcing.

---

## Project Overview

This wiki uses the **LLM Wiki pattern**: instead of re-deriving answers from raw documents on every
query (RAG), Claude incrementally builds and maintains a structured, interlinked collection of
markdown files. Knowledge is compiled once and kept current — not re-discovered on every question.

**Three layers:**

| Layer | Owner | Description |
|---|---|---|
| `chapters/` | Human | Raw, immutable source documents (articles, papers, transcripts, etc.) |
| `wiki/` | Claude | LLM-generated markdown: summaries, entity pages, concept pages, synthesis |
| `CLAUDE.md` | Co-evolved | This file — the schema that governs Claude's behavior |

---

## Directory Structure

```
/
├── CLAUDE.md               ← This file (schema & instructions)
├── chapters/                ← Raw source documents (read-only for Claude)
│   └── ...
└── wiki/
    ├── index.md            ← Content catalog: every page with a link and one-line summary
    ├── log.md              ← Append-only chronological event log
    ├── overview.md         ← High-level synthesis of the entire knowledge base
    ├── sources/            ← One summary page per ingested source
    ├── entities/           ← Pages for people, organizations, products, places, etc.
    ├── concepts/           ← Pages for ideas, themes, frameworks, terms
    ├── comparisons/        ← Side-by-side analyses and contrast pages
    └── queries/            ← Saved answers to notable past questions
```

---

## Wiki Conventions

### File naming
- Use `kebab-case.md` for all wiki files.
- Entity pages: `entities/firstname-lastname.md`, `entities/company-name.md`
- Concept pages: `concepts/topic-name.md`
- Source summaries: `sources/YYYY-MM-DD-short-title.md`
- Query pages: `queries/YYYY-MM-DD-short-description.md`

### Page structure
Every wiki page should begin with a YAML front matter block:

```yaml
---
title: "Page Title"
type: entity | concept | source | comparison | query | overview
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: [list of source filenames that informed this page]
related: [list of related wiki page paths]
---
```

Followed by:
1. **One-paragraph summary** — the most important thing to know about this topic.
2. **Body sections** — detailed content, organized with `##` headings.
3. **Connections** — a `## Connections` section with bullet-point links to related pages.
4. **Open questions** — a `## Open Questions` section listing unresolved gaps or contradictions.
5. **Sources** — a `## Sources` section listing the raw sources that informed this page.

### Cross-references
- Always link to related wiki pages using relative markdown links: `[Page Title](../entities/page.md)`
- When mentioning an entity or concept that has (or should have) its own page, link to it.
- Never leave an important term unlinked if a page for it exists.

### Contradictions
- When new information contradicts an existing claim, **do not silently overwrite**.
- Add a `> ⚠️ Contradiction noted: [source A] says X; [source B] says Y. Unresolved.` callout.
- Log the contradiction in `log.md`.

---

## Operations

### 1. Ingest

**Trigger:** Human drops a file into `chapters/` and says "ingest [filename]" or similar.

**Workflow:**

1. **Read** the source document in full.
2. **Discuss** with the human: what are the key takeaways? What's surprising, important, or
   contradictory to what's already in the wiki?
3. **Write a summary page** in `wiki/sources/` following the page structure above.
4. **Update entity pages** — for every person, organization, or named thing mentioned
   significantly, update or create their entity page.
5. **Update concept pages** — for every major idea, theme, or framework introduced or elaborated,
   update or create the concept page.
6. **Update `wiki/overview.md`** — revise the high-level synthesis if the new source shifts the
   picture meaningfully.
7. **Update `wiki/index.md`** — add the new summary page and any newly created pages.
8. **Append to `wiki/log.md`** — one entry per ingest (see log format below).

**Scope:** A single source might touch 5–15 wiki pages. That is expected and correct.

**Human involvement:** The default is to stay involved — discuss before writing, show summaries
before filing. If the human wants batch/silent ingest, they will say so.

---

### 2. Query

**Trigger:** Human asks a question.

**Workflow:**

1. **Read `wiki/index.md`** to identify relevant pages.
2. **Read those pages** in full.
3. **Synthesize** an answer with inline citations to wiki pages (and through them, to sources).
4. **Choose the right output format:**
   - General answer → prose with citations
   - Comparison → markdown table
   - Structured reference → new wiki page
   - Data/trends → suggest a chart (matplotlib) or table
   - Presentation → suggest Marp markdown slides
5. **Offer to file the answer** — if the answer is non-trivial (a comparison, an analysis,
   a discovered connection), ask the human whether to save it as a `wiki/queries/` page.
   Good answers should compound in the knowledge base, not disappear into chat history.
6. **Append to `wiki/log.md`** if the query produced a filed page.

---

### 3. Lint (Health Check)

**Trigger:** Human asks to "lint the wiki", "health check", or similar.

**Workflow — check for:**

- **Contradictions** — claims that conflict between pages; flag and note in those pages.
- **Stale claims** — assertions that newer sources have superseded; update with a note.
- **Orphan pages** — pages with no inbound links from other wiki pages; link them or flag.
- **Missing pages** — important entities or concepts mentioned in multiple pages but lacking
  their own page; create stubs or suggest creation.
- **Missing cross-references** — links that should exist but don't.
- **Data gaps** — topics where the wiki is thin and a web search or new source would help;
  surface these as suggestions.
- **Index drift** — pages that exist but are missing from `wiki/index.md`; add them.

**Output:** A lint report listing findings by category. Ask the human which items to address now.

---

## Index Format (`wiki/index.md`)

The index is a content catalog. Update it on every ingest.

```markdown
# Wiki Index

_Last updated: YYYY-MM-DD | Pages: N | Sources: N_

## Sources
| Page | Summary | Date | Source file |
|---|---|---|---|
| [Title](sources/filename.md) | One-line summary | YYYY-MM-DD | sources/filename |

## Entities
| Page | Summary |
|---|---|
| [Name](entities/name.md) | One-line summary |

## Concepts
| Page | Summary |
|---|---|
| [Concept](concepts/concept.md) | One-line summary |

## Comparisons
| Page | Summary |
|---|---|
| [Comparison](comparisons/name.md) | One-line summary |

## Queries
| Page | Summary | Date |
|---|---|---|
| [Query](queries/name.md) | One-line summary | YYYY-MM-DD |
```

---

## Log Format (`wiki/log.md`)

The log is append-only. **Never edit or delete past entries.** Add new entries at the top.

Each entry must start with `## [YYYY-MM-DD] <type> | <title>` for parseability.

```markdown
## [2026-04-02] ingest | Article Title
- Source: `sources/2026-04-02-article-title.md`
- Pages created: `entities/name.md`, `concepts/topic.md`
- Pages updated: `overview.md`, `index.md`, `entities/other.md`
- Key takeaway: One sentence on what this added to the wiki.
- Contradictions noted: (none) / (describe if any)

## [2026-04-02] query | Short question description
- Question: Full question asked.
- Pages read: list of pages consulted
- Output: prose / table / filed as `queries/filename.md`

## [2026-04-01] lint | Routine health check
- Orphans found: N
- Contradictions flagged: N
- New stubs created: N
- Summary: One sentence on overall wiki health.
```

**Tip for parsing the log from the terminal:**
```bash
grep "^## \[" wiki/log.md | head -10       # 10 most recent entries
grep "ingest" wiki/log.md                   # all ingests
grep "\[2026-04\]" wiki/log.md              # all entries from April 2026
```

---

## Overview Page (`wiki/overview.md`)

This is the single most important page in the wiki. It should:

- State the **central thesis or picture** that has emerged from all sources so far.
- Note the **key entities and concepts** (with links) that the wiki revolves around.
- Surface the **most important open questions** and unresolved contradictions.
- Be updated (not just appended to) with every significant ingest — it should always
  reflect the current synthesis, not just accumulate paragraphs.

---

## Behavioral Rules for Claude

1. **Claude writes the wiki; the human sources it.** Never ask the human to write wiki content.
   That is Claude's job. Claude does the summarizing, cross-referencing, filing, and bookkeeping.

2. **Read the index before answering queries.** Never answer from memory or prior context alone.
   Always ground answers in the current state of the wiki files.

3. **One ingest at a time by default.** Process sources one at a time and discuss with the human
   before filing. Switch to batch mode only if the human explicitly requests it.

4. **Never modify `sources/`.** That directory is read-only. Claude may read from it but must
   never write, rename, or delete anything in it.

5. **Never silently overwrite.** When new information contradicts old claims, flag the
   contradiction explicitly rather than silently replacing the old text.

6. **File valuable answers.** When a query produces a non-trivial answer — a comparison, an
   analysis, a synthesis — offer to file it in `wiki/queries/`. Good thinking shouldn't
   disappear into chat history.

7. **Keep `index.md` and `log.md` current.** These two files must be updated on every ingest
   and on every query that produces a filed page. They are the navigation backbone of the wiki.

8. **Prefer depth over breadth in entity and concept pages.** A page should synthesize
   everything the wiki knows about a topic — not just summarize the last source. Pull in
   context from earlier sources and cross-references.

9. **Surface open questions.** Every page should end with open questions — things the wiki
   doesn't know yet, gaps worth investigating, follow-up sources to seek. The wiki should
   guide future exploration, not just record the past.

10. **Co-evolve this schema.** If a workflow isn't working, a new page type is needed, or a
    convention should change, surface the suggestion to the human. This file should be updated
    to reflect what actually works for this wiki and domain.

---

## Getting Started (First Session)

If the wiki is empty, begin by:

1. Creating the directory structure listed above.
2. Creating a minimal `wiki/index.md` (empty tables, date set to today).
3. Creating a minimal `wiki/log.md` with a single initialization entry.
4. Creating a stub `wiki/overview.md` — one paragraph describing what this wiki will cover,
   based on what the human tells you.
5. Asking the human: "What's the first source you'd like to ingest?"

---

