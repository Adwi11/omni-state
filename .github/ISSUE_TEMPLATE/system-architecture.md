# 🏗️ System Architecture: Omni-State Protocol Implementation

## 🎯 Overview

**Omni-State Protocol** - A proactive, cross-platform AI memory layer that manages the "Current State of Truth" through hybrid Graph-Vector RAG architecture.

---

## 🏛️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BROWSER EXTENSION (Plasmo)                        │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────────┐│
│  │ DOM Watcher  │  │ Sidebar UI   │  │  Truth Nudges / Ghost Assistant    ││
│  └──────┬───────┘  └──────────────┘  └────────────────────────────────────┘│
└─────────┼───────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLOUDFLARE WORKERS (Go → Wasm)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │ /ingest API  │  │ /search API  │  │ /mcp API     │  │ Conflict Engine │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘  └────────┬────────┘ │
└─────────┼──────────────────┼────────────────────────────────────┼──────────┘
          │                  │                                     │
          ▼                  │                                     │
┌─────────────────────┐      │       ┌────────────────────────────────────────┐
│  PYTHON ML SERVICE  │◄─────┼──────►│           CLOUDFLARE KV               │
│  (FastAPI / Modal)  │      │       │    (Pending Resolution Queue)         │
│  ┌───────────────┐  │      │       └────────────────────────────────────────┘
│  │ Embeddings    │  │      │
│  │ Generator     │  │      │
│  ├───────────────┤  │      │
│  │ LLM Extractor │  │      │
│  │ (Fact Parser) │  │      │
│  ├───────────────┤  │      │
│  │ Batch Jobs    │  │      │
│  └───────┬───────┘  │      │
└──────────┼──────────┘      │
           │                 │
           ▼                 ▼
┌─────────────────────┐   ┌─────────────────────┐
│  QDRANT (Vector DB) │   │  NEO4J (Graph DB)   │
│  ├─ Semantic Search │   │  ├─ SPO Triples     │
│  ├─ Fuzzy Recall    │   │  ├─ Fact Versioning │
│  └─ Metadata Filter │   │  └─ Relationship    │
└─────────────────────┘   └─────────────────────┘
```

---

## 📦 Component Breakdown

### 1. 🦀 Go Backend (Cloudflare Workers + Wasm)
**Path:** `/backend`

| Module | Purpose |
|--------|---------|
| `cmd/worker/main.go` | Entry point using `syumai/workers` |
| `internal/truth` | Conflict detection & resolution state machine |
| `internal/mcp` | Model Context Protocol for Claude/ChatGPT |
| `internal/router` | Request routing & validation |

**Key Constraint:** No `net/http` — use `syumai/workers` fetch handler only.

---

### 2. 🐍 Python ML Microservice (NEW - Optimization)
**Path:** `/ml-service`

**Rationale:** Python excels at ML/AI tasks with superior library support. Separating concerns improves performance and maintainability.

**Deployment Options:**
- **Modal** (serverless Python, ideal for GPU workloads)
- **FastAPI on Fly.io / Railway**
- **Cloudflare Workers Python (beta)**

| Module | Purpose |
|--------|---------|
| `embeddings/generator.py` | Generate vectors using `sentence-transformers` |
| `extractor/fact_parser.py` | LLM-based fact extraction (langchain/llamaindex) |
| `batch/reindex.py` | Bulk re-embedding jobs |
| `api/main.py` | FastAPI endpoints |

**Python Dependencies:**
```
sentence-transformers>=2.2.0
langchain>=0.1.0
qdrant-client>=1.7.0
neo4j>=5.0.0
fastapi>=0.109.0
uvicorn>=0.27.0
pydantic>=2.0.0
```

---

### 3. 🌐 Browser Extension (Plasmo)
**Path:** `/extension`

| File | Purpose |
|------|---------|
| `background.ts` | DOM mutation observer, context detection |
| `sidebar.tsx` | Truth Nudges UI, Ghost Assistant |
| `content.ts` | Page injection, input field monitoring |

---

### 4. 💾 Data Layer

| Service | Purpose | Hosting |
|---------|---------|---------|
| **Qdrant** | Vector similarity search | Qdrant Cloud |
| **Neo4j** | Graph relationships, fact versioning | Neo4j Aura |
| **Cloudflare KV** | Resolution queue, caching | Cloudflare |

---

## 🔄 Optimized Data Flow

### Flow 1: Smart Ingestion (with Python ML)
```
User → Go API → Python ML Service → [Qdrant + Neo4j] → Response
         │              │
         │              ├─ Extract Facts (LLM)
         │              └─ Generate Embeddings
         │
         └─ Conflict Detection (Go State Machine)
```

### Flow 2: Proactive Search
```
Extension → Go API → Qdrant (Vector Search) → Truth Nudge
                          │
                          └─ Similarity threshold > 0.85
```

---

## 📋 Implementation Tasks

### Phase 1: Foundation
- [ ] Initialize Go backend with `syumai/workers` template
- [ ] Set up Python ML service with FastAPI
- [ ] Configure Qdrant Cloud instance
- [ ] Configure Neo4j Aura instance
- [ ] Set up Cloudflare Workers project

### Phase 2: Core Logic
- [ ] Implement `Fact` struct and validation
- [ ] Build embedding generation pipeline (Python)
- [ ] Build LLM fact extraction (Python + langchain)
- [ ] Implement conflict detection state machine (Go)
- [ ] Create Neo4j schema and queries

### Phase 3: APIs
- [ ] `/ingest` - Raw text ingestion
- [ ] `/search` - Semantic search
- [ ] `/resolve` - Manual conflict resolution
- [ ] `/mcp/*` - Model Context Protocol endpoints

### Phase 4: Extension
- [ ] Plasmo project setup
- [ ] DOM mutation observer
- [ ] Sidebar UI with React
- [ ] Truth Nudge notification system

### Phase 5: Integration
- [ ] End-to-end testing
- [ ] Performance optimization
- [ ] Documentation

---

## 🧠 Key Data Structures

### Fact (Go)
```go
type Fact struct {
    ID        string    `json:"id"`
    Subject   string    `json:"subject"`
    Predicate string    `json:"predicate"`
    Object    string    `json:"object"`
    Verified  bool      `json:"verified"`
    Timestamp time.Time `json:"timestamp"`
    Source    string    `json:"source"`
    Embedding []float32 `json:"embedding,omitempty"`
}
```

### Conflict (Go)
```go
type Conflict struct {
    ID          string `json:"id"`
    ExistingFact Fact  `json:"existing"`
    NewFact      Fact  `json:"new"`
    Status       string `json:"status"` // PENDING, RESOLVED, MERGED
    ResolvedAt   *time.Time `json:"resolved_at,omitempty"`
}
```

### Embedding Request (Python)
```python
class EmbeddingRequest(BaseModel):
    text: str
    model: str = "all-MiniLM-L6-v2"
    
class FactExtractionRequest(BaseModel):
    raw_text: str
    context: Optional[str] = None
```

---

## 🔧 Tech Stack Summary

| Layer | Technology | Why |
|-------|------------|-----|
| API Gateway | Go + Wasm + Cloudflare Workers | Edge performance, low latency |
| ML Processing | Python + FastAPI | Best ML ecosystem, langchain support |
| Vector Store | Qdrant | Open source, metadata filtering |
| Graph Store | Neo4j | Battle-tested, Cypher queries |
| Extension | Plasmo (React + TS) | Modern DX, cross-browser |
| Queue | Cloudflare KV | Integrated, low latency |

---

## 🚀 Why This Architecture?

1. **Go at the Edge** - Sub-millisecond routing, compiled to Wasm
2. **Python for ML** - Access to transformers, langchain, better LLM tooling
3. **Hybrid RAG** - Vector for fuzzy recall + Graph for factual truth
4. **Proactive UX** - Ghost assistant that nudges without interrupting

---

## 🔑 Key Optimizations Added

### 1. Python ML Service Separation
- **Problem:** Go lacks mature ML libraries for embeddings/LLM extraction
- **Solution:** Dedicated Python microservice handles all ML workloads
- **Benefit:** Use `sentence-transformers`, `langchain`, native Qdrant/Neo4j clients

### 2. Async Embedding Pipeline
- **Problem:** Embedding generation is CPU/GPU intensive
- **Solution:** Queue-based async processing with Modal or dedicated workers
- **Benefit:** Non-blocking ingestion, better UX

### 3. Batch Re-indexing Support
- **Problem:** Model upgrades require re-embedding all data
- **Solution:** Python batch jobs with progress tracking
- **Benefit:** Zero-downtime model migrations

### 4. Source Attribution
- **Problem:** Facts need provenance for trust scoring
- **Solution:** Added `Source` field to Fact struct
- **Benefit:** Better conflict resolution, audit trail

---

## Labels
`architecture` `system-design` `ml` `infrastructure`

