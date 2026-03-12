# Semantic Scholar API Reference

## Paper Search Endpoint

```
GET https://api.semanticscholar.org/graph/v1/paper/search
```

### Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `query` | Natural language search query | required |
| `fields` | Comma-separated fields to return | `paperId,title` |
| `limit` | Results per page (max 100) | 10 |
| `offset` | Pagination offset | 0 |
| `fieldsOfStudy` | Filter by field (e.g. `Computer Science`) | none |
| `year` | Year filter: `2024`, `2020-2024`, `-2024`, `2020-` | none |

### Recommended fields

```
title,authors,year,abstract,url,citationCount,externalIds,publicationVenue,openAccessPdf
```

### Full example

```
https://api.semanticscholar.org/graph/v1/paper/search?query=LLM+code+quality&fields=title,authors,year,abstract,url,citationCount,externalIds,publicationVenue&limit=100&year=2024-2026&fieldsOfStudy=Computer%20Science
```

## Response Structure

```json
{
  "total": 1234,
  "offset": 0,
  "next": 100,
  "data": [
    {
      "paperId": "abc123",
      "title": "Paper Title",
      "authors": [
        {"authorId": "1", "name": "John Doe"},
        {"authorId": "2", "name": "Jane Smith"}
      ],
      "year": 2024,
      "abstract": "Abstract text...",
      "url": "https://www.semanticscholar.org/paper/abc123",
      "citationCount": 42,
      "externalIds": {
        "DOI": "10.1145/3639478.3639800",
        "ArXiv": "2401.12345",
        "ACM": "3639478.3639800"
      },
      "publicationVenue": {
        "id": "...",
        "name": "ICSE 2024",
        "type": "conference"
      },
      "openAccessPdf": {
        "url": "https://arxiv.org/pdf/2401.12345",
        "status": "GREEN"
      }
    }
  ]
}
```

**Extraction notes:**
- `authors[i].name` — full name string (e.g. `"John Doe"`)
- `externalIds.DOI` — use for canonical link (`https://doi.org/<DOI>`)
- `externalIds.ArXiv` — cross-reference with arXiv results for deduplication
- `abstract` may be `null` for some papers even when requested — handle gracefully

## Rate Limits

- **Unauthenticated:** ~1 request per second; session-based limits may kick in sooner if many requests are made in short succession (HTTP 429). After a 429, wait **at least 30–60 seconds** before retrying.
- **With API key (free):** Much higher limits — get one at https://www.semanticscholar.org/product/api#api-key-form  
  Add header: `x-api-key: <YOUR_KEY>`

**Fleet strategy:** when running multiple queries in parallel, stagger requests by at least 1 second each, or use a free API key to avoid 429 errors.

## Advantages over arXiv

- Covers **ACM, IEEE, Springer, Elsevier** (not just arXiv preprints)
- Provides **citation counts** for impact assessment
- Returns **open access PDF links** when available
- Supports **year range** filtering natively (`year=2024-2026`)
- Broader coverage of **published conference and journal papers**

## Combining with arXiv

Strategy: use arXiv for the latest preprints; use Semantic Scholar for published, peer-reviewed work. Combine both, then deduplicate by title — the same paper often appears in both (arXiv preprint + published version).

When deduplicating: prefer the entry with a DOI (published) over the arXiv-only entry.
