---
title: "Building a Clinical Trials Intelligence Dashboard"
date: 2026-04-19
categories: ["data", "nextjs", "pinecone"]
---

ClinicalTrials.gov is one of the richest publicly available datasets in healthcare. It contains over 500,000 studies spanning decades of research across every therapeutic area imaginable. The data includes everything from study design and eligibility criteria to sponsor information, site locations with geocoordinates, and free-text descriptions of what each trial is actually investigating. It's a goldmine.

And yet, I couldn't find a single good web app that lets you explore it properly. The official site has a search interface, but it's focused on finding individual studies — not on understanding the landscape. If you want to know which conditions have the most active trials right now, or which sponsors are running the largest studies, or how trial volume has changed over time, you're out of luck. You'd need to download a bulk export and wrangle it yourself. I wanted something better: an interactive dashboard with charts, maps, filters, and an AI-powered search that lets you describe what you're looking for in plain English. So I built [Clinical Trials Intelligence](https://clinicaltrials.app).

## The Data Pipeline

The project is split into two parts: a Python data pipeline that fetches and normalises the data, and a Next.js web app that visualises it.

The pipeline uses the [ClinicalTrials.gov v2 API](https://clinicaltrials.gov/data-api/api) to page through study records. The API returns deeply nested JSON — a single study can have dozens of nested modules for identification, status, design, eligibility, contacts, locations, and more. The pipeline flattens this into a normalised relational schema and loads it into PostgreSQL.

```python
def flatten_study(study: Study) -> FlatStudyRow:
    nct_id = extract_nct_id(study)
    id_mod = safe_get(study, ["protocolSection", "identificationModule"]) or {}
    status_mod = safe_get(study, ["protocolSection", "statusModule"]) or {}
    design_mod = safe_get(study, ["protocolSection", "designModule"]) or {}
    eligibility_mod = safe_get(study, ["protocolSection", "eligibilityModule"]) or {}

    return {
        "nct_id": nct_id,
        "org_study_id": safe_get(id_mod, ["orgStudyIdInfo", "id"]),
        "brief_title": id_mod.get("briefTitle"),
        "overall_status": status_mod.get("overallStatus"),
        "start_date": normalize_date(safe_get(status_mod, ["startDateStruct", "date"])),
        "phases": ",".join(design_mod.get("phases", []) or []),
        "enrollment_count": safe_get(design_mod, ["enrollmentInfo", "count"]),
        # ... more fields
    }
```

A `safe_get` helper handles the nested access gracefully — the API response is inconsistent about which fields are present, so defensive traversal is essential.

### The Schema

I chose a normalised schema with junction tables rather than a denormalised wide table. The core tables are:

| Table | Description |
|-------|-------------|
| `studies` | Core study record keyed by `nct_id` |
| `study_text` | Full-text fields (title, summary, description, eligibility) — 1:1 with studies |
| `locations` | Deduplicated facilities with geocoordinates |
| `conditions` | Deduplicated condition names |
| `sponsors` | Deduplicated sponsor organisations |
| `icd10_chapters` | 22 ICD-10-CM chapter definitions |

Each entity table (locations, conditions, sponsors) is linked to studies via junction tables (`study_locations`, `study_conditions`, `study_sponsors`) with composite primary keys. Deduplication happens on insert using normalised (lowercased, trimmed) text, so "Pfizer" and "pfizer" don't create separate records.

The reason for keeping `study_text` separate from `studies` is practical: the text fields are large (detailed descriptions can be thousands of characters) and are only needed for full-text search and the Pinecone upsert. Keeping them out of the main `studies` table means the analytical queries that power the dashboard charts don't have to scan through megabytes of text.

### Condition Classification

One of the more interesting parts of the pipeline is automatic condition classification. ClinicalTrials.gov doesn't categorise conditions in any standardised way — you get free-text strings like "Type 2 Diabetes Mellitus" or "Non-Small Cell Lung Cancer" or "Healthy Volunteers". To make the conditions page useful, I needed to map these to a standard taxonomy.

I chose ICD-10-CM chapters as the classification target. There are 22 chapters covering broad categories like "Neoplasms", "Diseases of the circulatory system", and "Mental, behavioral and neurodevelopmental disorders". The classification uses embedding similarity: I wrote prototype descriptions for each chapter (including example conditions), embedded them with `sentence-transformers/all-mpnet-base-v2`, and then for each condition, I compute cosine similarity against all 22 chapter embeddings and assign the best match.

```python
class ConditionClassifier:
    def __init__(self, device: str = "cpu"):
        self.model = SentenceTransformer(MODEL_NAME, device=device)
        prototype_texts = [ch[2] for ch in CHAPTER_PROTOTYPES]
        self.chapter_embeddings = self.model.encode(
            prototype_texts, normalize_embeddings=True, convert_to_numpy=True
        )

    def classify_batch(self, conditions: list[str]) -> list[tuple[str, float]]:
        condition_embeddings = self.model.encode(
            conditions, normalize_embeddings=True, convert_to_numpy=True
        )
        similarities = np.dot(condition_embeddings, self.chapter_embeddings.T)
        results = []
        for i in range(len(conditions)):
            best_idx = int(np.argmax(similarities[i]))
            best_score = float(similarities[i][best_idx])
            _, chapter_name, _ = CHAPTER_PROTOTYPES[best_idx]
            results.append((chapter_name, best_score))
        return results
```

It runs on Apple Silicon via MPS and processes about 10,000 conditions in 10 minutes. The results are stored back in the `conditions` table with a confidence score, so the dashboard can show an ICD-10 chapter breakdown chart.

This approach is deliberately simple. You could use a fine-tuned classifier or even an LLM for this, but embedding similarity with good prototype descriptions is surprisingly effective and requires zero training data. The prototypes include example conditions for each chapter, which helps the model disambiguate edge cases.

## The Web App

The dashboard is a Next.js 15 app using the App Router with server components. It's deployed on Vercel and backed by a Neon-hosted PostgreSQL database — the same schema the pipeline writes to.

### Pages

There are five pages, each focused on a different dimension of the data:

- **Studies**: Trial volume over time (area chart), phase distribution (pie chart), and status breakdown (bar chart)
- **Sponsors**: Top sponsors by trial count and patient enrolment, sponsor class distribution (industry vs. academic vs. government)
- **Conditions**: ICD-10 chapter breakdown and most-studied conditions
- **Locations**: An interactive world bubble map showing trial density by country, plus a top countries bar chart
- **AI Search**: Natural language vector search

All pages except AI Search support filtering by date range, trial phase, and status (all / completed / active). The filters use [nuqs](https://nuqs.47ng.com/) to store state in URL search params, so views are shareable and bookmarkable. This was a deliberate choice over React state — if someone finds an interesting view, they can share the URL and it will render exactly the same.

The charts use Highcharts with a custom dark theme. The map uses Highcharts Maps with bubble series, where each bubble is sized proportionally to the number of trials in that country.

### Server Components and Data Fetching

One of the nice things about the App Router is that the data fetching happens entirely on the server. Each page is a server component that calls query functions directly — no API routes needed for the dashboard data. The query functions build SQL with dynamic WHERE clauses based on the active filters:

```typescript
function buildStatusClause(status?: string) {
  if (!status || status === 'all') return { clause: '', params: [] };
  if (status === 'completed') {
    return { clause: "AND s.overall_status = $1", params: ['COMPLETED'] };
  }
  if (status === 'active') {
    return {
      clause: "AND s.overall_status = ANY($1::text[])",
      params: [['RECRUITING', 'ACTIVE_NOT_RECRUITING', 'ENROLLING_BY_INVITATION']],
    };
  }
  return { clause: '', params: [] };
}
```

The parameterisation is a bit manual — each clause tracks its own params and the calling function stitches them together with incrementing `$N` placeholders — but it's straightforward and avoids any ORM overhead. For a read-heavy analytics dashboard, raw SQL is hard to beat.

## AI Search

The AI Search page is the feature I'm most pleased with. Instead of trying to match keywords against study titles, you describe what you're looking for in natural language:

> "Find phase 3 clinical trials for type 2 diabetes in adults over 50 that involve GLP-1 receptor agonists and are currently recruiting"

Behind the scenes, this uses Pinecone's [integrated inference](https://docs.pinecone.io/guides/inference/understanding-inference) feature. The Pinecone index is configured with the `llama-text-embed-v2` model (1024 dimensions, cosine similarity), which means Pinecone generates the embeddings server-side. This is a significant simplification: I don't need to run an embedding model in the API route or manage a separate embedding service. I just send the text and Pinecone handles the rest.

The data that gets embedded is a concatenation of all text fields for each study — title, summary, detailed description, and eligibility criteria. This gives the vector search a rich signal to match against:

```python
def build_text(row: dict) -> str:
    parts: list[str] = []
    if row.get("official_title"):
        parts.append(f"Title: {row['official_title']}")
    if row.get("brief_summary"):
        parts.append(f"Summary: {row['brief_summary']}")
    if row.get("detailed_description"):
        parts.append(f"Description: {row['detailed_description']}")
    if row.get("eligibility_criteria"):
        parts.append(f"Eligibility: {row['eligibility_criteria']}")
    return "\n\n".join(parts)
```

The upsert script handles rate limiting with exponential backoff and supports resuming from a specific `nct_id` in case the process gets interrupted — which it will, because upserting hundreds of thousands of records takes a while.

On the query side, the API route is simple: take the user's query, send it to Pinecone's `searchRecords` endpoint, get back the top 10 matching `nct_id`s with scores, then hydrate them from PostgreSQL:

```typescript
const searchResponse = await index.searchRecords({
  query: {
    topK: 10,
    inputs: { text: query.trim() },
  },
});

const nctIds = hits.map((hit) => hit._id as string);
const result = await pool.query(
  `SELECT s.nct_id, st.official_title, st.brief_summary, s.overall_status, ...
   FROM studies s JOIN study_text st ON s.nct_id = st.nct_id
   WHERE s.nct_id = ANY($1::text[])`,
  [nctIds]
);
```

There's a minimum query length of 50 characters. This is intentional — short keyword queries don't work well with semantic search. You need enough context for the embedding model to understand the intent. "diabetes" is too vague; "phase 3 interventional studies investigating SGLT2 inhibitors for diabetic kidney disease" gives the model something meaningful to work with.

### Why Pinecone with Integrated Inference

I considered a few alternatives for the vector search:

1. **pgvector**: Would keep everything in PostgreSQL and avoid a separate service. But generating embeddings would require running a model in the API route, adding latency and complexity. Also, pgvector's ANN performance degrades at scale without careful tuning.

2. **Self-hosted embedding + Pinecone**: More control over the embedding model, but adds operational complexity. I'd need to manage model loading, batching, and version consistency between the upsert pipeline and the query path.

3. **Pinecone with integrated inference**: The simplest option. Embeddings are generated server-side by Pinecone, so the upsert script just sends raw text and the API route just sends the query string. No model management at all. The trade-off is less control over the embedding model — you're limited to what Pinecone offers — but `llama-text-embed-v2` is a strong model and more than adequate for this use case.

For a project like this where the priority is getting something working and deployed rather than squeezing out the last few percentage points of retrieval accuracy, option 3 was the clear winner.

## Tech Stack Summary

- **Data pipeline**: Python, `psycopg`, ClinicalTrials.gov v2 API, `sentence-transformers`, Pinecone SDK
- **Database**: PostgreSQL (Neon) — normalised schema with junction tables
- **Web app**: Next.js 15, TypeScript, Tailwind CSS, Highcharts, Radix UI
- **Vector search**: Pinecone with integrated inference (`llama-text-embed-v2`)
- **Package manager**: Bun
- **Deployment**: Vercel
- **Fonts**: Bricolage Grotesque (display), Libre Franklin (body)

## Useful Links

- [Clinical Trials Intelligence](https://clinicaltrials.app)
- [ClinicalTrials.gov v2 API](https://clinicaltrials.gov/data-api/api)
- [Pinecone Integrated Inference](https://docs.pinecone.io/guides/inference/understanding-inference)
- [Neon — Serverless PostgreSQL](https://neon.tech)
- [nuqs — URL State Management](https://nuqs.47ng.com/)
- [sentence-transformers](https://www.sbert.net/)
