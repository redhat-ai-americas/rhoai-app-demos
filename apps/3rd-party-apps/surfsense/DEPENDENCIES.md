# SurfSense Dependencies and Architecture

## Component Overview

| Component | Purpose | Type | License | Required |
|-----------|---------|------|---------|----------|
| **SurfSense** | Main application | Container | Check upstream | ✅ Yes |
| **PostgreSQL + pgvector** | Vector database | Container | PostgreSQL License | ✅ Yes |
| **Redis** | Cache & sessions | Container | BSD-3-Clause | ✅ Yes |
| **Docling** | Document parsing | Container | MIT | ⚠️ Recommended |
| **SearXNG** | Web search | Container | AGPL-3.0 | 🔧 Optional |
| **OAuth Proxy** | OpenShift auth | Sidecar | - | ✅ Yes |
| **RHOAI** | LLM & embeddings | External | - | ✅ Yes |

## Full Dependency Tree

```
SurfSense Deployment
│
├─ Core Application Stack
│  ├─ SurfSense Container (ghcr.io/modsetter/surfsense:latest)
│  │  ├─ Frontend (Port 3000)
│  │  ├─ Backend API (Port 8000)
│  │  └─ Electric-SQL (Port 5133)
│  │
│  └─ OAuth Proxy Sidecar (quay.io/openshift/origin-oauth-proxy:4.15)
│     └─ OpenShift Authentication (Port 8443)
│
├─ Data Layer
│  ├─ PostgreSQL 16 with pgvector (pgvector/pgvector:pg16)
│  │  ├─ Vector storage for embeddings
│  │  ├─ Full-text search (tsquery/tsvector)
│  │  └─ Document metadata storage
│  │  └─ Storage: 20Gi PVC
│  │
│  └─ Redis 7 (redis:7-alpine)
│     └─ Session caching and job queue
│
├─ Processing Services
│  ├─ Docling (quay.io/docling-project/docling-serve:latest)
│  │  ├─ PDF parsing
│  │  ├─ Office documents (DOCX, PPTX, XLSX)
│  │  ├─ HTML, Markdown, CSV
│  │  └─ Image OCR
│  │
│  └─ SearXNG (searxng/searxng:latest)
│     └─ Metasearch aggregation (70+ search engines)
│
├─ AI/ML Layer (External)
│  └─ RHOAI InferenceServices
│     ├─ LLM Model (e.g., granite-7b-instruct)
│     └─ Embedding Model (e.g., granite-embedding)
│
└─ Storage
   ├─ SurfSense PVC (10Gi) - Application data
   └─ PostgreSQL PVC (20Gi) - Database persistence
```

## Container Images

### Required Images

```yaml
# SurfSense main application
ghcr.io/modsetter/surfsense:latest

# PostgreSQL with pgvector extension
pgvector/pgvector:pg16

# Redis cache
redis:7-alpine

# OpenShift OAuth proxy
quay.io/openshift/origin-oauth-proxy:4.15
```

### Optional Images

```yaml
# Docling document parser (recommended)
quay.io/docling-project/docling-serve:latest
# Alternative: ghcr.io/docling-project/docling-serve:latest

# SearXNG web search (optional)
searxng/searxng:latest
```

## Resource Requirements

### Minimum Requirements (for testing)

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit | Storage |
|-----------|-------------|----------------|-----------|--------------|---------|
| SurfSense | 1 core | 2Gi | 2 cores | 4Gi | 10Gi |
| PostgreSQL | 500m | 1Gi | 1 core | 2Gi | 20Gi |
| Redis | 250m | 512Mi | 500m | 1Gi | - |
| Docling | 1 core | 2Gi | 2 cores | 4Gi | - |
| SearXNG | 500m | 512Mi | 1 core | 1Gi | - |
| **Total** | **3.25 cores** | **6Gi** | **6.5 cores** | **12Gi** | **30Gi** |

### Recommended Requirements (for production)

| Component | CPU Request | Memory Request | CPU Limit | Memory Limit | Storage |
|-----------|-------------|----------------|-----------|--------------|---------|
| SurfSense | 2 cores | 4Gi | 4 cores | 8Gi | 50Gi |
| PostgreSQL | 1 core | 2Gi | 2 cores | 4Gi | 100Gi |
| Redis | 500m | 1Gi | 1 core | 2Gi | - |
| Docling | 2 cores | 4Gi | 4 cores | 8Gi | - |
| SearXNG | 1 core | 1Gi | 2 cores | 2Gi | - |
| **Total** | **6.5 cores** | **12Gi** | **13 cores** | **24Gi** | **150Gi** |

## Network Dependencies

### Internal (Cluster) Communication

```
SurfSense Backend → PostgreSQL (postgresql-service:5432)
SurfSense Backend → Redis (redis-service:6379)
SurfSense Backend → Docling (docling-service:5001)
SurfSense Backend → SearXNG (searxng-service:8080)
SurfSense Backend → RHOAI InferenceServices (e.g., granite-7b-predictor.demo.svc.cluster.local/v1)

OAuth Proxy → SurfSense Frontend (localhost:3000)
User → OpenShift Route → OAuth Proxy (8443)
```

### External Dependencies

1. **RHOAI InferenceServices** (required)
   - Cross-namespace access to models
   - Requires RoleBinding for ServiceAccount

2. **Internet Access** (optional)
   - SearXNG needs internet to query search engines
   - Can be disabled if no web search needed

3. **OpenShift OAuth** (required)
   - For user authentication via OAuth proxy
   - Uses OpenShift identity provider

## Database Schema

### PostgreSQL with pgvector

**Extensions Required:**
- `vector` - For pgvector support (embeddings storage)
- Built-in full-text search (`tsquery`, `tsvector`)

**Tables** (created automatically by SurfSense):
- Documents (with vector embeddings)
- Chunks (with vector embeddings)
- Users
- Sessions
- Search indices

## Document Processing Pipeline

```
User uploads document
       ↓
SurfSense Backend receives file
       ↓
Send to Docling Service (HTTP POST /v1/convert)
       ↓
Docling parses document (PDF/DOCX/etc.)
       ↓
Returns structured content (JSON/Markdown)
       ↓
SurfSense chunks the content
       ↓
Send chunks to RHOAI Embedding Model
       ↓
Receive embeddings (vectors)
       ↓
Store in PostgreSQL with pgvector
       ↓
Index for full-text search (tsvector)
```

## RAG Query Pipeline

```
User asks a question
       ↓
Optional: SearXNG web search (if enabled)
       ↓
Generate query embedding (RHOAI Embedding Model)
       ↓
Hybrid search in PostgreSQL:
  ├─ Vector similarity (pgvector cosine distance)
  └─ Full-text search (PostgreSQL tsquery)
       ↓
Reciprocal Rank Fusion (combine results)
       ↓
Two-tiered retrieval:
  ├─ Document-level (entire documents)
  └─ Chunk-level (specific passages)
       ↓
Send context + question to RHOAI LLM
       ↓
Return response with citations
```

## Security Considerations

### OpenShift Security

- **SecurityContext**: Uses `fsGroup: 0` for storage access
- **SCC**: Requires `anyuid` SCC for PostgreSQL
- **ServiceAccount**: Dedicated SA with OAuth annotations
- **RBAC**: Cross-namespace access to RHOAI InferenceServices

### Network Security

- **TLS**: Route uses edge termination
- **OAuth**: OpenShift OAuth proxy for authentication
- **Internal**: All service-to-service traffic is HTTP (within cluster)
- **Secrets**: Database passwords and API keys stored in Kubernetes Secret

### Data Privacy

- **Docling**: Processes documents locally (no external API)
- **SearXNG**: Self-hosted (no external search API calls)
- **Embeddings**: Processed by RHOAI (stays in cluster)
- **Data**: All data stored in cluster (PostgreSQL PVC)

## Alternative Configurations

### 1. Use External Services

Replace bundled services with external ones:

```yaml
postgresql:
  enabled: false  # Use external PostgreSQL
  # Configure DATABASE_URL manually

redis:
  enabled: false  # Use external Redis
  # Configure REDIS_URL manually

docling:
  enabled: false  # Use external Docling service
  # Or use UNSTRUCTURED or LLAMACLOUD with API key

searxng:
  enabled: false  # Disable web search
  # Or use Tavily/Linkup with API key
```

### 2. Minimal Deployment (Just SurfSense + RHOAI)

Deploy only SurfSense and use external services for everything:

- External PostgreSQL with pgvector
- External Redis
- Skip Docling (use UNSTRUCTURED cloud API)
- Skip SearXNG (disable web search)

**Pros**: Lower resource usage in cluster
**Cons**: Requires external service management, potential API costs

### 3. All-in-One with GPU

Add GPU support for Docling for faster document processing:

```yaml
docling:
  resources:
    limits:
      nvidia.com/gpu: "1"
  image:
    tag: "cu128"  # CUDA 12.8 build
```

## Comparison with AnythingLLM

| Feature | SurfSense | AnythingLLM |
|---------|-----------|-------------|
| Vector DB | PostgreSQL + pgvector | Built-in LanceDB |
| Document Parsing | Docling (external) | Built-in |
| Web Search | SearXNG (optional) | Not included |
| Architecture | Microservices | Monolithic |
| RAG Approach | Two-tiered hybrid | Single-tier |
| Browser Extension | ✅ Yes | ❌ No |
| Dependencies | More complex | Simpler |
| Resource Usage | Higher | Lower |
| Scalability | Better (separate services) | Limited |

## Open Source vs. Commercial Alternatives

### Fully Open Source Stack (Current)
- ✅ SurfSense (open-source)
- ✅ Docling (MIT)
- ✅ SearXNG (AGPL-3.0)
- ✅ PostgreSQL (PostgreSQL License)
- ✅ Redis (BSD-3-Clause)
- 💰 Total cost: $0 (just infrastructure)

### With Commercial Services
- SurfSense + Unstructured API ($0-$200/month)
- SurfSense + LlamaCloud ($0-$100/month)
- SurfSense + Tavily ($0-$50/month for 1000 requests)
- SurfSense + Linkup ($0-$40/month)

## Future Enhancements

Potential additions to the deployment:

1. **Firecrawl** - Open-source web scraping (alternative to SearXNG)
2. **GPU support** - For Docling acceleration
3. **Monitoring** - Prometheus + Grafana
4. **Backup** - Velero for PostgreSQL backups
5. **Scaling** - HorizontalPodAutoscaler for SurfSense
6. **Multi-LLM** - Support multiple RHOAI models
7. **LangSmith** - LLM observability integration
