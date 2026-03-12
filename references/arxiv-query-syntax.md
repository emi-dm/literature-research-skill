# arXiv Query Syntax Reference

## Base URL

```
https://export.arxiv.org/api/query?search_query=<QUERY>&start=0&max_results=50&sortBy=submittedDate&sortOrder=descending
```

## Field Prefixes

| Prefix | Searches in |
|--------|------------|
| `ti:`  | Title |
| `au:`  | Author |
| `abs:` | Abstract |
| `co:`  | Comment |
| `jr:`  | Journal reference |
| `cat:` | Subject category |
| `rn:`  | Report number |
| `all:` | All fields (default) |

## Boolean Operators

**Critical:** In arXiv API query strings, `+` is decoded as a **space**, and a space between terms means **OR**.  
So `all:LLM+code+quality` is parsed as `all:LLM OR code OR quality` — almost certainly not what you want.

**Always use `+AND+` between every `field:term` pair:**

```
# ✅ Correct — AND between field:term pairs
all:LLM+AND+all:code+AND+all:quality+AND+cat:cs.SE

# ✅ Correct — phrase search (URL-encode the quotes)
ti:%22code+generation%22+AND+cat:cs.SE

# ✅ Correct — multiple field:term with AND
abs:LLM+AND+abs:code+AND+abs:quality+AND+cat:cs.SE

# ❌ Wrong — + is OR, not AND
all:LLM+code+quality+AND+cat:cs.SE
# parsed as: all:LLM OR code OR quality AND cat:cs.SE
```

The feed `<title>` in the response always echoes back the parsed query — useful for debugging.

**Operators (always surround with `+`):**
- `+AND+` — both terms required
- `+OR+` — either term
- `+ANDNOT+` — exclude term

## Categories for Software Engineering Research

| Category | Name |
|----------|------|
| `cs.SE`  | Software Engineering |
| `cs.PL`  | Programming Languages |
| `cs.LG`  | Machine Learning |
| `cs.AI`  | Artificial Intelligence |
| `cs.CR`  | Cryptography and Security |
| `cs.NE`  | Neural and Evolutionary Computing |
| `cs.IR`  | Information Retrieval |

## Sorting Options

- `sortBy=submittedDate&sortOrder=descending` — newest first (recommended)
- `sortBy=relevance` — most relevant first
- `sortBy=lastUpdatedDate` — recently updated first

## Pagination

Use `start=0`, `start=50`, `start=100` etc. with `max_results=50`.
The response includes `<opensearch:totalResults>` to know how many pages exist.

## Parsing the Atom XML Response

Each paper is wrapped in `<entry>...</entry>`. Key fields:

```xml
<entry>
  <id>http://arxiv.org/abs/2401.12345v1</id>
  <title>Paper Title Here</title>
  <summary>Abstract text...</summary>
  <published>2024-01-23T10:00:00Z</published>
  <updated>2024-01-23T10:00:00Z</updated>
  <author><name>John Doe</name></author>
  <author><name>Jane Smith</name></author>
  <category term="cs.SE" scheme="..."/>
  <link rel="alternate" type="text/html" href="http://arxiv.org/abs/2401.12345v1"/>
  <link rel="related" type="application/pdf" href="http://arxiv.org/pdf/2401.12345v1"/>
</entry>
```

Extract with string operations or regex since full XML parsers may not be available:
- Title: text between `<title>` and `</title>` (skip the first `<title>` which is the feed title)
- Abstract: text between `<summary>` and `</summary>`
- Year: first 4 characters of `<published>`
- URL: `href` attribute of `<link rel="alternate">`
- arXiv ID: last path segment of the URL (e.g. `2401.12345v1` → `2401.12345`)

## Example Queries for LLM / Software Engineering Research

Each term needs its own `field:` prefix when using AND. For phrase searches, URL-encode quotes as `%22`.

```
# General LLM code quality
abs:LLM+AND+abs:code+AND+abs:quality+AND+cat:cs.SE

# Phrase search: "code generation" in title
ti:%22code+generation%22+AND+cat:cs.SE

# Security vulnerabilities in LLM code
abs:LLM+AND+abs:vulnerability+AND+(cat:cs.SE+OR+cat:cs.CR)

# Code maintainability / technical debt
abs:LLM+AND+abs:maintainability+AND+cat:cs.SE
abs:LLM+AND+abs:%22technical+debt%22+AND+cat:cs.SE

# Testing LLM-generated code
abs:LLM+AND+abs:testing+AND+cat:cs.SE

# Benchmarks and evaluation
abs:LLM+AND+abs:benchmark+AND+abs:%22code+generation%22+AND+cat:cs.SE

# Specific models
abs:GPT+AND+abs:%22code+quality%22+AND+cat:cs.SE
ti:%22GitHub+Copilot%22+AND+cat:cs.SE
abs:CodeLlama+AND+abs:quality+AND+cat:cs.SE
```
