# Lodenix

**The living intelligence layer for AI tools.** Real-world signal, by domain, ready for any AI prompt.

Lodenix turns real-time web data into structured, scored, domain-specific knowledge — served via clean REST endpoints. Your AI tools get what's actually working in the world right now, without you building any RAG infrastructure.

---

## The Problem

Every AI tool you build needs current, domain-specific ground truth. Without it, your AI is guessing based on stale training data. Lodenix gives your AI what's actually working in the world right now.

## The Solution

Zyte crawls curated, high-signal sources per domain on a schedule. An AI scoring layer extracts and ranks only what's valuable. That knowledge is embedded, stored per collection, and served via REST endpoints your tools can call with a single POST request.

---

## Architecture

```
Web Sources → Zyte Crawler → Extraction & Cleaning → AI Scoring → Vector Store → REST API → Your AI Tool
```

| Stage | Component | Description |
|-------|-----------|-------------|
| 01 | **Ingestion** | Zyte scrapes targeted, curated source lists per domain on a fixed schedule |
| 02 | **Processing** | Strip noise, extract signal, normalize into consistent schema |
| 03 | **Intelligence** | AI scores every chunk for value (0–1), tags intents, deduplicates, weights by recency |
| 04 | **Storage** | Embeddings stored per domain collection with auto-pruning on each crawl cycle |
| 05 | **API Layer** | Clean REST endpoints per domain — query in, ranked knowledge out |
| 06 | **Consumer** | Your tool injects knowledge directly into the AI prompt |

---

## Quickstart

Make your first knowledge request in under 60 seconds.

### cURL

```bash
curl -X POST https://api.lodenix.dev/v1/knowledge/sales \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "cold email openers that get replies",
    "top_k": 5,
    "recency_bias": 0.8
  }'
```

### Python

```python
import requests

response = requests.post(
    "https://api.lodenix.dev/v1/knowledge/sales",
    headers={"Authorization": "Bearer YOUR_API_KEY"},
    json={
        "query": "cold email openers that get replies",
        "top_k": 5,
        "recency_bias": 0.8
    }
)

knowledge = response.json()["results"]
context = "\n\n".join([r["content"] for r in knowledge])
```

### Node.js

```javascript
const res = await fetch('https://api.lodenix.dev/v1/knowledge/sales', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    query: 'cold email openers that get replies',
    top_k: 5,
    recency_bias: 0.8
  })
})

const { results } = await res.json()
const context = results.map(r => r.content).join('\n\n')
```

---

## Authentication

All API requests must include your API key as a Bearer token in the `Authorization` header:

```
Authorization: Bearer lx_live_xxxxxxxxxxxxxxxxxxxx
```

> **Keep your key secret.** Never expose it in client-side code. Use environment variables.

---

## API Reference

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/knowledge/{domain}` | Query a knowledge domain |
| `GET`  | `/v1/domains` | List all available domains |
| `GET`  | `/v1/domains/{domain}/status` | Check freshness and chunk count |
| `GET`  | `/v1/health` | API health check |

### Request Parameters

POST body for `/v1/knowledge/{domain}`:

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `query` | `string` | Yes | — | Natural language query for knowledge retrieval |
| `top_k` | `integer` | No | `5` | Number of results to return (max: `20`) |
| `recency_bias` | `float` | No | `0.6` | 0–1. Higher values weight newer knowledge more |
| `min_score` | `float` | No | `0.4` | Minimum relevance score threshold 0–1 |
| `format` | `string` | No | `"chunks"` | `"chunks"` or `"prompt_ready"` — pre-formatted for injection |

### Response Format

```json
{
  "domain": "sales",
  "query": "cold email openers that get replies",
  "freshness": "2025-05-02T09:41:00Z",
  "result_count": 5,
  "results": [
    {
      "id": "lx_chnk_a8f3c",
      "content": "Pattern interrupt openers outperform subject lines by 34% in B2B outreach...",
      "score": 0.91,
      "value": 0.87,
      "source": "sales-hacker-forum",
      "tags": ["cold-email", "openers", "b2b"],
      "crawled_at": "2025-05-02T08:10:00Z"
    }
  ]
}
```

| Field | Description |
|-------|-------------|
| `results[].content` | The extracted knowledge chunk. Ready to inject into a prompt. |
| `results[].score` | Semantic relevance to your query (0–1). Higher is more relevant. |
| `results[].value` | AI-assigned quality score — how genuinely useful this chunk is. |
| `freshness` | When this domain was last crawled. Use to decide if you need fresher data. |

---

## Knowledge Domains

| Domain | Endpoint | Description | Crawl Interval |
|--------|----------|-------------|----------------|
| Sales | `/v1/knowledge/sales` | Objection handling, cold outreach, closing tactics | Every 2h |
| Trading | `/v1/knowledge/trading` | Market sentiment, signals, macro trends | Every 1h |
| Content | `/v1/knowledge/content` | Viral hooks, formats, trending topics | Every 3h |
| Copywriting | `/v1/knowledge/copy` | Offer structures, headlines, CTAs, ad angles | Every 6h |

### Freshness & TTL

Each domain has a crawl schedule tuned to how fast that topic moves:

```http
GET /v1/domains/sales/status
```

```json
{
  "domain": "sales",
  "last_crawl": "2025-05-02T09:41:00Z",
  "next_crawl": "2025-05-02T11:41:00Z",
  "crawl_interval": "2h",
  "chunk_count": 2418,
  "status": "healthy"
}
```

---

## Prompt Injection Pattern

The recommended way to use Lodenix results in your AI tools:

```python
async def build_sales_prompt(user_query):
    # 1. Fetch fresh knowledge
    knowledge = await lodenix.query(
        domain="sales",
        query=user_query,
        top_k=5
    )

    # 2. Build context block
    context = "\n\n".join([
        f"INSIGHT: {r['content']}"
        for r in knowledge["results"]
    ])

    # 3. Inject into system prompt
    system_prompt = f"""
You are an expert sales AI.

CURRENT KNOWLEDGE (what's working right now):
{context}

Use the above insights to inform your responses.
Always apply what's actually working, not generic advice.
    """

    return system_prompt
```

> **Pro tip:** Use `format: "prompt_ready"` in your request and we pre-format the context block for you — ready to paste directly into your system prompt.

---

## Custom Domains

On Pro and Enterprise plans you can define your own knowledge domains — supply your source list and crawl schedule, we handle the rest.

> Custom domain configuration coming soon via the dashboard. For early access contact `api@lodenix.dev`

---

## Tech Stack

- **Crawling:** Zyte Pro
- **Processing:** RAGFlow / R2R
- **Storage:** pgvector
- **API:** FastAPI
- **AI Scoring:** Claude / GPT-4

---

## Development

```bash
# Clone the repository
git clone https://github.com/Forge-Growth/lodenix.git

# Navigate to the project
cd lodenix

# (more instructions coming soon)
```

---

## License

Proprietary. All rights reserved.

---

## Contact

- Website: [lodenix.dev](https://lodenix.dev)
- Email: `api@lodenix.dev`
- GitHub: [Forge-Growth/lodenix](https://github.com/Forge-Growth/lodenix)
