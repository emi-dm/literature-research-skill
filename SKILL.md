---
name: literature-research
description: Systematic literature research skill for finding, filtering, deduplicating and reporting academic papers. Specialised in arXiv (cs.SE and related), Semantic Scholar and CrossRef. Use when the user asks to search for papers, build a literature survey, investigate the state of the art, or generate a bibliography report.
---

# Literature Research Skill

A systematic workflow for conducting academic literature reviews, covering arXiv, Semantic Scholar, and CrossRef. Produces structured Markdown reports grouped by sub-topic, with deduplication across queries.

## When to Use This Skill

**Use when the user says things like:**
- "Busca papers en arXiv sobre…"
- "Investiga el estado del arte de…"
- "Genera un reporte de literatura sobre…"
- "Find papers about…"
- "Search ACM / arXiv / Semantic Scholar for…"
- "Do a literature survey on…"

## Core Workflow

Follow these steps exactly in this order:

### Step 1 — Gather preferences

Ask (or infer from context) the following parameters before searching:
- **Topic**: The subject or research question.
- **Year range**: Default to the last 3 years if not specified.
- **Max papers per query**: Default to 50.
- **Sources**: Default to arXiv + Semantic Scholar. Mention CrossRef for ACM/IEEE papers.
- **Sub-topics / sections**: How the user wants the report grouped (e.g. Security, Maintainability, Testing).
- **Output language for the report**: usually Spanish summaries + English paper metadata.

### Step 2 — Build query set

Design **5–10 diverse queries** covering the topic from different angles:
- Include synonyms and related terms.
- Vary abstraction level: specific (e.g. "GPT-4 code smell detection") and general (e.g. "LLM software quality").
- For arXiv: target categories like `cs.SE`, `cs.PL`, `cs.LG` as appropriate.

### Step 3 — Execute searches in parallel

Use `/fleet` to run multiple searches simultaneously. Each parallel agent should:
1. Fetch papers using the arXiv API (see below).
2. Fetch papers using the Semantic Scholar API (see below).
3. Return results as a structured JSON list.

### Step 4 — Deduplicate

Merge all results. Deduplicate by normalising titles (lowercase, strip punctuation) and comparing. Keep the entry with the most metadata (abstract, DOI, URL).

Use a session SQL table to track seen titles:
```sql
CREATE TABLE IF NOT EXISTS lit_papers (
    id TEXT PRIMARY KEY,
    title TEXT,
    authors TEXT,
    year INTEGER,
    abstract TEXT,
    url TEXT,
    doi TEXT,
    citations INTEGER DEFAULT 0,
    venue TEXT,
    source TEXT,
    section TEXT
);
```

### Step 5 — Filter

Apply:
- Year range filter.
- Relevance check: does the title/abstract mention the topic keywords?
- For LLM-related surveys: require at least one of: GPT, Copilot, LLM, large language model, CodeLlama, Claude, DeepSeek, Gemini, Codex in title or abstract.

### Step 6 — Generate report

Produce a structured Markdown report (see Report Format section below).

---

## API Reference

### arXiv API

```
GET https://export.arxiv.org/api/query
  ?search_query=<QUERY>
  &start=0
  &max_results=<N>
  &sortBy=submittedDate
  &sortOrder=descending
```

**Query syntax:**
- `all:keyword` — search all fields
- `ti:keyword` — title only
- `abs:keyword` — abstract only
- `cat:cs.SE` — restrict to category
- `AND`, `OR`, `ANDNOT` — Boolean operators (surround with `+`: `+AND+`, `+OR+`, `+ANDNOT+`)
- **`+` between terms = space = OR** — to AND two terms, use `+AND+` between separate `field:term` pairs
- Phrase search: URL-encode quotes as `%22`, e.g. `ti:%22code+generation%22`

**Example:**
```
https://export.arxiv.org/api/query?search_query=abs:LLM+AND+abs:code+AND+abs:quality+AND+cat:cs.SE&start=0&max_results=50&sortBy=submittedDate&sortOrder=descending
```

**Debugging tip:** the `<title>` element at the top of the Atom feed echoes back the parsed query — use it to verify your query was interpreted correctly.

**Response:** Atom XML. Extract per `<entry>`:
- `<title>` → paper title (skip the first `<title>` which belongs to the feed, not a paper)
- `<author><name>` → authors (collect all)
- `<summary>` → abstract
- `<published>` → e.g. `2024-03-15T12:00:00Z` → year = first 4 chars
- `<link rel="alternate" href="...">` → URL
- arXiv ID: last segment of the URL path

**Important categories for Software Engineering:**
| Category | Description |
|----------|-------------|
| `cs.SE`  | Software Engineering |
| `cs.PL`  | Programming Languages |
| `cs.LG`  | Machine Learning |
| `cs.CR`  | Cryptography and Security |
| `cs.AI`  | Artificial Intelligence |

---

### Semantic Scholar API

```
GET https://api.semanticscholar.org/graph/v1/paper/search
  ?query=<QUERY>
  &fields=title,authors,year,abstract,url,citationCount,externalIds,publicationVenue
  &limit=<N>
  &year=<YEAR_START>-<YEAR_END>
  &fieldsOfStudy=Computer Science   ← optional
```

No API key required for basic use, but **rate limits are strict** (~1 req/sec). On HTTP 429, wait 30–60 seconds before retrying. For parallel fleet searches, get a free API key at https://www.semanticscholar.org/product/api#api-key-form and add header `x-api-key: <KEY>`.

**Example:**
```
https://api.semanticscholar.org/graph/v1/paper/search?query=LLM+generated+code+quality&fields=title,authors,year,abstract,url,citationCount,externalIds,publicationVenue&limit=50&year=2024-2026&fieldsOfStudy=Computer%20Science
```

**Response:** JSON. Structure:
```json
{
  "data": [
    {
      "paperId": "...",
      "title": "...",
      "authors": [{"authorId": "...", "name": "Full Name"}],
      "year": 2024,
      "abstract": "..." ,
      "url": "https://www.semanticscholar.org/paper/...",
      "citationCount": 42,
      "externalIds": {"DOI": "10.xxxx/...", "ArXiv": "2401.12345"},
      "publicationVenue": {"name": "ICSE 2024"}
    }
  ]
}
```

**Extraction notes:** `authors[i].name` is a full-name string. `abstract` may be `null` — handle gracefully. Use `externalIds.ArXiv` to cross-reference and deduplicate against arXiv results.

---

### CrossRef API (for ACM/IEEE papers)

```
GET https://api.crossref.org/works
  ?query=<QUERY>
  &filter=from-pub-date:<YEAR>,type:journal-article
  &rows=<N>
  &select=DOI,title,author,published,URL,container-title,is-referenced-by-count
```

**Example:**
```
https://api.crossref.org/works?query=LLM+code+quality&filter=from-pub-date:2024,type:journal-article&rows=50&select=DOI,title,author,published,URL,container-title,is-referenced-by-count
```

Use `mailto=<email>` param if available for polite pool (faster responses).

**Response schema — critical field types (all verified):**
```json
{
  "message": {
    "items": [
      {
        "DOI": "10.xxxx/yyyyy",
        "title": ["Paper Title Here"],
        "author": [
          {"given": "Jane", "family": "Smith", "sequence": "first", "affiliation": []}
        ],
        "published": {"date-parts": [[2024]]},
        "URL": "https://doi.org/10.xxxx/yyyyy",
        "container-title": ["Journal or Conference Name"],
        "is-referenced-by-count": 5
      }
    ]
  }
}
```

**Extraction notes:**
- `title` is an **array** → use `title[0]`
- `container-title` is an **array** → use `container-title[0]`
- `author[i].given` + `" "` + `author[i].family` → full name (not `author[i].name`)
- `published["date-parts"][0][0]` → year as integer
- `abstract` is **not reliably returned** by CrossRef — omit from `select` or treat as optional

---

## Report Format

Produce the report in Markdown. Adapt sections to the user's sub-topics.

```markdown
# <Topic> — Estado del Arte <YEAR_START>–<YEAR_END>

> Generado: <DATE> · **<N> papers únicos** · Fuentes: arXiv, Semantic Scholar

## Resumen Ejecutivo

<3–5 bullet points with the main findings synthesised from the abstracts>

## Metodología

- **Queries ejecutadas:** <list>
- **Fuentes:** arXiv (cs.SE, cs.LG), Semantic Scholar
- **Filtros:** años <RANGE>, LLM-relacionado
- **Total papers brutos:** <N> · **Tras deduplicación:** <N>

## Tabla de Contenidos

- [Section 1](#section-1) (N papers)
- [Section 2](#section-2) (N papers)
...

## <Section 1>

### 1. <Paper Title> (<YEAR>)

**Autores:** <Authors>  
**Venue:** <Journal/Conference>  
**Fuente:** arXiv / Semantic Scholar  
**URL:** <URL>  
**Citas:** <N>  

**Resumen:** <1–2 sentence synthesis in Spanish>

---

### 2. ...

## Referencias

| # | Título | Autores | Año | Fuente | DOI/URL |
|---|--------|---------|-----|--------|---------|
| 1 | ...    | ...     | ... | ...    | ...     |
```

---

## Quality Checks

Before delivering the report, verify:
1. **All papers are within the year range** — filter any outlier.
2. **LLM relevance** (if applicable) — title or abstract must mention an LLM.
3. **No duplicate titles** — normalise before final count.
4. **URLs are valid arXiv / DOI links** — not placeholder text.
5. **Resumen Ejecutivo reflects actual findings** — not generic statements.

---

## Fleet Strategy

For large surveys (>5 queries), use this fleet pattern:

```
Fleet:
  Agent A → arXiv queries 1–3
  Agent B → arXiv queries 4–6
  Agent C → Semantic Scholar queries 1–3
  Agent D → Semantic Scholar queries 4–6
→ Main agent: merge, deduplicate, build report
```

Each fleet agent should return a JSON block like:
```json
{
  "source": "arXiv",
  "query": "LLM code smell detection cs.SE",
  "papers": [...]
}
```

---

## Common Mistakes to Avoid

- ❌ **Do NOT use Google Scholar** — blocks automated access.
- ❌ **Do NOT scrape dl.acm.org directly** — returns 403; use CrossRef or Semantic Scholar instead.
- ❌ **Do NOT guess abstracts** — only include abstracts retrieved from the API.
- ❌ **Do NOT count duplicates** in the final paper count.
- ✅ **Always provide direct links** to the paper (arXiv abs page or DOI).

---

## References

See `references/` for:
- `arxiv-query-syntax.md` — Complete arXiv query language guide
- `semantic-scholar-api.md` — Semantic Scholar API tips and rate limits
- `report-template.md` — Full blank report template

See `examples/` for:
- `llm-code-quality-report.md` — Example completed report (475 papers, 14 sections)
