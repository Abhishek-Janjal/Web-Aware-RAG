# Web-Aware RAG Engine

> A production-ready, scalable Retrieval-Augmented Generation system with asynchronous web content ingestion and semantic querying capabilities.

## Demo Video

**[Watch 8-Minute System Walkthrough →](https://drive.google.com/file/d/11oqWkZgkA4j2Kp_hS6YuXbDVWMDSyxHR/view?usp=sharing)**

Demonstrates:
- Async URL ingestion with immediate 202 response
- Worker processing pipeline (fetch → clean → chunk → embed)
- Real-time job status tracking
- Semantic search with cited answer generation
- Vector storage and metadata inspection

---

## System Architecture

```mermaid
flowchart LR
    classDef store fill:#fff,stroke:#333,stroke-width:1px
    classDef svc fill:#fff,stroke:#333,stroke-width:1px,rx:6,ry:6
    classDef decision fill:#fff,stroke:#333,stroke-dasharray: 5 5,rx:6,ry:6

    C[Client]:::svc
    A[FastAPI API]:::svc
    D{URL Exists?}:::decision
    R1[Redis Queue]:::store
    W[Celery Workers]:::svc
    E[OpenAI Embeddings]:::svc
    SQL[PostgreSQL Metadata]:::store
    V[ChromaDB Vectors]:::store
    R2[Redis Queue]:::store
    L[GPT-4o-mini]:::svc
    S{Context > Window?}:::decision
    SUM[Summarizer]:::svc

    C -- "POST /ingest-url" --> A
    A -- "202 + job_id" --> C
    
    A --> D
    D -- "No" --> R1
    D -- "Yes" --> C
    R1 --> W
    W --> E
    E --> V
    W --> SQL
    
    C -- "POST /query" --> A
    A --> R2
    R2 --> V
    V --> S
    S -- "No" --> L
    S -- "Yes" --> SUM
    SUM --> L
    L -- "Answer + Citations" --> C
```

### Architecture Highlights

**Async-First Design**
- Immediate 202 acknowledgment on URL submission
- Background processing via Celery workers
- Redis-backed job queue with retry logic

**Dual-Store Pattern**
- **ChromaDB**: Vectors + metadata for semantic search
- **PostgreSQL**: Job status, URL tracking, ACID guarantees

**Smart Deduplication**
- URL normalization with hash-based lookup
- HTTP header validation (Last-Modified/ETag)
- Prevents redundant downloads and processing

---

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **API Framework** | FastAPI | Async support, automatic OpenAPI docs, type validation |
| **Task Queue** | Celery + Redis | Enterprise-grade retries, horizontal scaling, monitoring |
| **Vector DB** | ChromaDB | Lightweight, metadata filtering, easy deployment |
| **Metadata DB** | PostgreSQL | ACID compliance, concurrent writes, advanced indexing |
| **Web Scraping** | trafilatura | Clean extraction, language detection, fallback support |
| **Embeddings** | OpenAI text-embedding-3-large | 3072 dims, state-of-the-art semantic quality |
| **LLM** | GPT-4o-mini | Cost-effective, fast, maintains quality for RAG |
| **ORM** | SQLAlchemy | Database abstraction, maintainability |

### Key Design Decisions

**Why Background Processing?**
- Web scraping is slow (3-10s per page)
- Network failures require retry logic
- Prevents API timeouts and poor UX
- Enables horizontal scaling of workers for production workloads

**Why Separate Vector + Relational Stores?**
- ChromaDB optimized for cosine similarity search
- PostgreSQL handles complex queries and transactions
- Clear separation enables independent scaling strategies

**Why Context Compression?**
- Reduces token overhead for large result sets
- Improves relevance by focusing on key information
- Maintains quality while staying within context windows

---

## Data Model

### Entity-Relationship Diagram

```mermaid
erDiagram
    urls ||--o{ ingestions : tracks
    urls {
      uuid id PK
      text url UNIQUE
      text url_hash UNIQUE
      timestamptz created_at
      jsonb submit_meta
    }
    
    ingestions {
      uuid id PK
      uuid url_id FK
      text status "pending|running|completed|failed"
      timestamptz started_at
      timestamptz finished_at
      text error
      int chunk_count
    }
    
    chunks {
      uuid id PK
      uuid url_id FK
      text content
      int position
      jsonb metadata
    }
    
    vectors {
      uuid chunk_id FK
      float[] embedding
      jsonb metadata
    }
```

### Schema Design

**PostgreSQL (Metadata)**
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE urls (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  url TEXT NOT NULL UNIQUE,
  url_hash TEXT NOT NULL UNIQUE,
  submit_meta JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE ingestions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  url_id UUID REFERENCES urls(id) ON DELETE CASCADE,
  status TEXT CHECK (status IN ('pending','running','completed','failed')),
  started_at TIMESTAMPTZ,
  finished_at TIMESTAMPTZ,
  error TEXT,
  chunk_count INT
);

CREATE INDEX idx_ingestions_url_id ON ingestions(url_id);
CREATE INDEX idx_ingestions_status ON ingestions(status);
```

**ChromaDB (Vectors)**
- Collection: `chunks_v1`
- Metadata per vector: `{url, title, chunk_id, lang, site, timestamp}`
- Enables filtering: `{"site": "docs.example.com", "lang": "en"}`

---

## API Reference

### 1. Ingest URL

Submit a URL for background processing.

**Endpoint:** `POST /ingest-url`

**Request Body:**
```json
{
  "url": "https://example.com/article",
  "force": false,
  "tags": ["docs", "api"]
}
```

**Response:** `202 Accepted`
```json
{
  "job_id": "1e7c4a8b-...",
  "url_id": "a9c1f2d3-...",
  "status_url": "/status/1e7c4a8b-..."
}
```

**cURL Example:**
```bash
curl -X POST http://localhost:8000/ingest-url \
  -H 'Content-Type: application/json' \
  -H 'X-API-Key: your-key' \
  -d '{
    "url": "https://nordvpn.com/blog/vpn-for-ipad/",
    "tags": ["security"]
  }'
```

---

### 2. Check Job Status

Poll ingestion progress.

**Endpoint:** `GET /status/{job_id}`

**Response:** `200 OK`
```json
{
  "job_id": "1e7c4a8b-...",
  "status": "completed",
  "url": "https://example.com/article",
  "metrics": {
    "chunk_count": 23,
    "content_bytes": 41234,
    "processing_time_ms": 4521
  },
  "error": null
}
```

**cURL Example:**
```bash
curl http://localhost:8000/status/1e7c4a8b-...
```

---

### 3. List Ingested URLs

View all processed URLs with filters.

**Endpoint:** `GET /urls`

**Query Parameters:**
- `status` (optional): `pending|running|completed|failed`
- `limit` (default: 50): Results per page
- `offset` (default: 0): Pagination offset

**Response:** `200 OK`
```json
{
  "items": [
    {
      "url": "https://example.com/article",
      "status": "completed",
      "last_ingested_at": "2025-10-29T10:30:00Z",
      "chunk_count": 23
    }
  ],
  "total": 1,
  "limit": 50,
  "offset": 0
}
```

**cURL Example:**
```bash
curl "http://localhost:8000/urls?status=completed&limit=10"
```

---

### 4. Query Knowledge Base

Ask questions with semantic search.

**Endpoint:** `POST /query`

**Request Body:**
```json
{
  "query": "What are the security benefits of VPNs on mobile?",
  "top_k": 8,
  "filters": {
    "site": ["nordvpn.com"],
    "lang": ["en"]
  },
  "answer": {
    "max_tokens": 400,
    "temperature": 0.2
  }
}
```

**Response:** `200 OK`
```json
{
  "answer": "VPNs provide encryption and privacy protection...",
  "citations": [
    {
      "chunk_id": "abc123...",
      "url": "https://nordvpn.com/blog/vpn-for-ipad/",
      "title": "VPN for iPad - Security Guide",
      "text": "Encryption prevents data interception...",
      "relevance_score": 0.89
    }
  ],
  "latency_ms": 312,
  "chunks_retrieved": 8
}
```

**cURL Example:**
```bash
curl -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "How does encryption work?",
    "top_k": 5,
    "filters": {"lang": ["en"]}
  }'
```

---

## Processing Pipeline

### Ingestion Workflow

```
1. URL Submission → Normalize & deduplicate
2. Job Creation → Insert into PostgreSQL, return 202
3. Queue Task → Push to Redis, picked by worker
4. Fetch Content → HTTP GET with timeout/redirects
5. Extract Text → trafilatura cleaning
6. Chunk Text → Recursive splitter (800 tokens, 100 overlap)
7. Generate Embeddings → OpenAI API (batched)
8. Store Vectors → Upsert to ChromaDB with metadata
9. Update Status → Mark completed in PostgreSQL
```

### Query Workflow

```
1. Embed Query → text-embedding-3-large
2. Vector Search → ChromaDB cosine similarity, top-k
3. Apply Filters → Site, language, date metadata
4. Check Context → If >20 chunks, trigger summarizer
5. Build Prompt → Numbered citations + context
6. LLM Generate → GPT-4o-mini with temp=0.2
7. Return Result → Answer + citations + metadata
```

### Chunking Strategy

Production RAG systems require careful chunking to balance context richness with retrieval precision:

- **Target size:** 800 tokens (~3000 chars)
- **Overlap:** 100 tokens to preserve context
- **Boundaries:** Respect headings, paragraphs
- **Metadata:** Preserve position, section headers

---

## Configuration

### Environment Setup

Copy `.env.example` to `.env` and configure:

```bash
cp .env.example .env
```

**`.env.example`:**
```dotenv
# API Configuration
API_HOST=0.0.0.0
API_PORT=8000
API_KEY=your-secret-key-here
CORS_ORIGINS=http://localhost:3000,https://app.example.com

# Redis & Celery
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=${REDIS_URL}
CELERY_RESULT_BACKEND=${REDIS_URL}

# PostgreSQL
DATABASE_URL=postgresql+psycopg2://postgres:postgres@postgres:5432/ragengine

# ChromaDB
CHROMA_HOST=chroma
CHROMA_PORT=8000
CHROMA_COLLECTION=chunks_v1

# OpenAI
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-large
LLM_MODEL=gpt-4o-mini
MAX_TOKENS=400
TEMPERATURE=0.2

# Fetch Limits
FETCH_TIMEOUT_SECONDS=20
MAX_PAGE_BYTES=5242880
ALLOWED_DOMAINS=  # Comma-separated, empty = all allowed
```

---

## Quick Start

### Prerequisites

- Docker & Docker Compose
- OpenAI API key
- 4GB RAM minimum

### 1. Clone Repository

```bash
git clone https://github.com/akesh1235/web-aware-rag-challenge.git
cd web-aware-rag-challenge
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env and add your OPENAI_API_KEY
```

### 3. Start Services

```bash
docker compose up --build
```

Services will start on:
- **API:** http://localhost:8000
- **API Docs:** http://localhost:8000/docs
- **ChromaDB:** http://localhost:8001
- **PostgreSQL:** localhost:5432

### 4. Test the System

**Ingest a URL:**
```bash
curl -X POST http://localhost:8000/ingest-url \
  -H 'Content-Type: application/json' \
  -d '{"url": "https://en.wikipedia.org/wiki/Artificial_intelligence"}'
```

**Check status (use job_id from above):**
```bash
curl http://localhost:8000/status/{job_id}
```

**Query the knowledge:**
```bash
curl -X POST http://localhost:8000/query \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "What is artificial intelligence?",
    "top_k": 5
  }'
```

---

## Docker Compose

**`docker-compose.yml`:**
```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - redis
      - postgres
      - chroma
    command: uvicorn api.main:app --host 0.0.0.0 --port 8000 --workers 2

  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile
    env_file: .env
    depends_on:
      - redis
      - postgres
      - chroma
    command: celery -A workers.celery_app worker -Q ingest -l info -c 2

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ragengine
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  chroma:
    image: ghcr.io/chroma-core/chroma:latest
    ports:
      - "8001:8000"
    environment:
      - IS_PERSISTENT=TRUE
    volumes:
      - chroma_data:/chroma/chroma

volumes:
  redis_data:
  postgres_data:
  chroma_data:
```

---

## Repository Structure

```
web-aware-rag-challenge/
├── api/
│   ├── main.py              # FastAPI app initialization
│   ├── deps.py              # Dependency injection
│   ├── schemas.py           # Pydantic models
│   └── routers/
│       ├── ingest.py        # POST /ingest-url
│       ├── query.py         # POST /query
│       ├── status.py        # GET /status/:id
│       └── urls.py          # GET /urls
├── core/
│   ├── config.py            # Environment configuration
│   ├── chunking.py          # Text splitting logic
│   ├── embedding.py         # OpenAI embeddings
│   ├── llm.py               # GPT-4o-mini generation
│   ├── extractor.py         # Web scraping (trafilatura)
│   └── retrieval.py         # ChromaDB search
├── workers/
│   ├── celery_app.py        # Celery configuration
│   └── tasks.py             # Background job definitions
├── db/
│   ├── models.py            # SQLAlchemy models
│   ├── session.py           # Database connection
│   └── migrations/          # Alembic migrations
├── vector/
│   ├── chroma_client.py     # ChromaDB client
│   └── __init__.py
├── tests/
│   ├── test_ingest.py       # Ingestion tests
│   ├── test_query.py        # Query tests
│   └── fixtures.py          # Test data
├── docker/
│   └── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env.example
└── README.md
```

---

## Testing

### Run Test Suite

```bash
# Install dependencies
pip install -r requirements.txt
pip install pytest pytest-asyncio

# Run tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=. --cov-report=html
```

### Test Coverage

- URL validation and normalization
- Deduplication logic
- Chunking edge cases
- Embedding generation
- Vector search with filters
- Citation formatting
- Error handling and retries

---

## Security

### Production Considerations

**Authentication**
- API key via `X-API-Key` header
- Rate limiting: 60 req/min per key (token bucket)
- CORS configuration for browser clients

**Content Safety**
- robots.txt respect (configurable)
- Domain allowlist/blocklist
- HTML sanitization
- Redirect limits (max 5)
- Timeout enforcement (20s default)

**Data Protection**
- Environment variables for secrets
- Docker secrets support
- No credentials in logs
- SQL injection protection (ORM)

---

## Monitoring & Reliability

### Observability

**Structured Logging**
```json
{
  "timestamp": "2025-10-29T10:30:45Z",
  "level": "INFO",
  "service": "worker",
  "job_id": "1e7c4a8b",
  "event": "ingestion_completed",
  "chunk_count": 23,
  "duration_ms": 4521
}
```

**Metrics Tracked**
- Ingestion success/failure rates
- Processing latency (p50, p95, p99)
- Vector search latency
- LLM token usage
- Queue depth

### Reliability Features

Production RAG systems require robust error handling and automated refresh pipelines:

**Retry Logic**
- Exponential backoff: 1s, 2s, 4s, 8s, 16s
- Max 5 retries per task
- Transient error handling (network, API rate limits)

**Deduplication**
- URL hash prevents duplicate ingestion
- Content checksum skips redundant updates
- ETag/Last-Modified for freshness checks

**Graceful Degradation**
- Return "insufficient context" if no relevant chunks
- Fallback to cached results on API failures
- Circuit breaker for external services

---

## Performance Optimization

### Scaling Strategies

**Horizontal Scaling**
- Add worker instances: `docker compose up --scale worker=4`
- ChromaDB server mode for distributed queries
- PostgreSQL read replicas for status queries

**Caching**
- Redis cache for frequent queries (TTL: 5min)
- Embedding cache to avoid re-processing
- Document hash to skip unchanged content

**Batch Processing**
- Embed chunks in batches of 100
- Bulk upsert to ChromaDB
- Parallel fetching for sitemap ingestion

### Resource Limits

| Resource | Development | Production |
|----------|-------------|------------|
| Worker Memory | 512MB | 2GB |
| API Memory | 256MB | 1GB |
| Worker Concurrency | 2 | 8-16 |
| Max Chunk Size | 800 tokens | 1000 tokens |
| Vector Dimensions | 3072 | 3072 |

---

## Advanced Features

### Context Compression

When queries retrieve >20 chunks:
1. **Summarizer Agent** condenses content
2. Preserves key facts and relationships
3. Reduces token overhead by ~60%
4. Maintains answer quality while fitting context windows

### Hybrid Search (Planned)

Combine semantic + keyword search:
```python
results = chroma.query(
    query_embeddings=[query_vector],
    where={"lang": "en"},
    where_document={"$contains": "encryption"}  # Keyword filter
)
```

### Multi-Hop Reasoning

For complex queries requiring multiple retrievals:
1. Initial retrieval identifies relevant domains
2. Follow-up query refines based on first results
3. Aggregate evidence across multiple chunks

---

## Troubleshooting

### Common Issues

**Worker not processing jobs**
```bash
# Check Redis connection
docker compose logs redis

# Verify worker is running
docker compose logs worker

# Restart worker
docker compose restart worker
```

**ChromaDB connection error**
```bash
# Check ChromaDB logs
docker compose logs chroma

# Verify port binding
curl http://localhost:8001/api/v1/heartbeat
```

**Database migration needed**
```bash
# Run migrations
docker compose exec api alembic upgrade head
```

---

## API Examples

### Python Client

```python
import requests

BASE_URL = "http://localhost:8000"

# Ingest URL
response = requests.post(
    f"{BASE_URL}/ingest-url",
    json={"url": "https://example.com"},
    headers={"X-API-Key": "your-key"}
)
job_id = response.json()["job_id"]

# Poll status
status = requests.get(f"{BASE_URL}/status/{job_id}").json()
print(f"Status: {status['status']}")

# Query
result = requests.post(
    f"{BASE_URL}/query",
    json={
        "query": "What is the main topic?",
        "top_k": 5
    }
).json()
print(f"Answer: {result['answer']}")
for cite in result['citations']:
    print(f"  - {cite['url']}")
```

### JavaScript Client

```javascript
const BASE_URL = 'http://localhost:8000';

// Ingest URL
const ingestResponse = await fetch(`${BASE_URL}/ingest-url`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-API-Key': 'your-key'
  },
  body: JSON.stringify({ url: 'https://example.com' })
});
const { job_id } = await ingestResponse.json();

// Query
const queryResponse = await fetch(`${BASE_URL}/query`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: 'What is this about?',
    top_k: 5
  })
});
const result = await queryResponse.json();
console.log(result.answer);
```

---

## License

MIT © 2025

---
