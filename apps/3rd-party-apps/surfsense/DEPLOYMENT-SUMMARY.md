# SurfSense Deployment - Summary

## What Was Created

A complete Helm chart deployment for **SurfSense** - an open-source alternative to NotebookLM and Perplexity, fully integrated with Red Hat OpenShift AI (RHOAI).

## 📁 File Structure

```
apps/3rd-party-apps/surfsense/
├── README.md                          # Complete documentation
├── QUICKSTART.md                      # Step-by-step deployment guide
├── DEPENDENCIES.md                    # Detailed dependency analysis
├── helm/
│   ├── Chart.yaml                     # Helm chart metadata
│   ├── values.yaml                    # Default configuration
│   ├── values-rhoai.yaml              # RHOAI-specific overrides
│   └── templates/
│       ├── namespace.yaml             # demo-apps namespace
│       ├── serviceaccount.yaml        # surfsense SA with OAuth
│       ├── rolebinding.yaml           # anyuid SCC binding
│       ├── cross-namespace-rolebinding.yaml  # RHOAI access
│       ├── secret.yaml                # Credentials and config
│       ├── pvc.yaml                   # SurfSense storage (10Gi)
│       ├── postgres-pvc.yaml          # PostgreSQL storage (20Gi)
│       ├── postgres-deployment.yaml   # PostgreSQL with pgvector
│       ├── postgres-service.yaml      # PostgreSQL service
│       ├── redis-deployment.yaml      # Redis cache
│       ├── redis-service.yaml         # Redis service
│       ├── docling-deployment.yaml    # Docling parser
│       ├── docling-service.yaml       # Docling service
│       ├── searxng-deployment.yaml    # SearXNG web search
│       ├── searxng-service.yaml       # SearXNG service
│       ├── deployment.yaml            # Main SurfSense app
│       ├── service.yaml               # SurfSense service
│       ├── oauth-proxy-service.yaml   # OAuth proxy service
│       └── route.yaml                 # OpenShift route
└── openshift/
    ├── kustomization.yaml             # Kustomize config
    └── route.yaml                     # Alternative route config
```

**Total**: 27 files, ~900 lines of YAML

## 🎯 Key Features Included

### Core Capabilities
- ✅ **RAG System** - Two-tiered (document + chunk level) with hybrid search
- ✅ **Document Parsing** - Docling for PDF, DOCX, PPTX, HTML, images, CSV
- ✅ **Web Search** - SearXNG for autonomous web searching (100% open-source)
- ✅ **Vector Database** - PostgreSQL with pgvector extension
- ✅ **Full-Text Search** - PostgreSQL native tsquery/tsvector
- ✅ **Browser Extension** - Chrome/Firefox extension (built into SurfSense)
- ✅ **RHOAI Integration** - LLM and embedding models from RHOAI
- ✅ **OAuth Authentication** - OpenShift OAuth proxy integration
- ✅ **Multi-LLM Support** - Compatible with any OpenAI-compatible API

### Deployment Pattern
- ✅ **Microservices Architecture** - Separate containers for each service
- ✅ **High Availability Ready** - Designed for scaling
- ✅ **Persistent Storage** - PVCs for data persistence
- ✅ **Security** - RBAC, SCC, secrets management
- ✅ **Observability** - Liveness and readiness probes

## 🔧 Components Deployed

| Component | Image | Purpose |
|-----------|-------|---------|
| **SurfSense** | `ghcr.io/modsetter/surfsense:latest` | Main application (frontend + backend) |
| **PostgreSQL** | `pgvector/pgvector:pg16` | Vector database with pgvector |
| **Redis** | `redis:7-alpine` | Cache and session storage |
| **Docling** | `quay.io/docling-project/docling-serve:latest` | Document parsing service |
| **SearXNG** | `searxng/searxng:latest` | Web search metasearch engine |
| **OAuth Proxy** | `quay.io/openshift/origin-oauth-proxy:4.15` | OpenShift authentication |

## 📊 Resource Requirements

### Minimum (Testing)
- **CPU**: 3.25 cores requested, 6.5 cores limit
- **Memory**: 6Gi requested, 12Gi limit
- **Storage**: 30Gi (10Gi app + 20Gi database)

### Recommended (Production)
- **CPU**: 6.5 cores requested, 13 cores limit
- **Memory**: 12Gi requested, 24Gi limit
- **Storage**: 150Gi (50Gi app + 100Gi database)

## 🚀 Quick Deploy

### Option 1: GitOps (Recommended)

```bash
# 1. Update RHOAI endpoints in values-rhoai.yaml
vi apps/3rd-party-apps/surfsense/helm/values-rhoai.yaml

# 2. Deploy via ArgoCD
oc apply -f gitops/apps/surfsense.yaml

# 3. Monitor deployment
oc get application surfsense -n openshift-gitops -w
```

**Benefits:**
- ✅ Automated sync from Git
- ✅ Proper dependency ordering with sync waves
- ✅ Self-healing and rollback capabilities
- ✅ Declarative and auditable

See **[GitOps Quick Reference](../../gitops/apps/SURFSENSE-QUICK-REFERENCE.md)** for commands.

### Option 2: Direct Helm

```bash
cd apps/3rd-party-apps/surfsense

# 1. Update RHOAI endpoints in values-rhoai.yaml
vi helm/values-rhoai.yaml

# 2. Deploy
helm install surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  -n demo-apps --create-namespace

# 3. Get route URL
oc get route surfsense -n demo-apps
```

## 🔑 Key Configuration Points

### Required Configuration
1. **RHOAI InferenceService URLs** - Update in `values-rhoai.yaml`:
   - LLM endpoint (e.g., granite-7b-predictor)
   - Embedding endpoint (e.g., granite-embedding-predictor)

### Optional Configuration
- **Authentication**: LOCAL (default) or GOOGLE OAuth
- **Web Search**: SearXNG (default), Tavily, Linkup, or disabled
- **Document Processing**: Docling (default), Unstructured, or LlamaCloud
- **Storage Classes**: Specify custom storage classes if needed
- **Resources**: Adjust CPU/memory based on workload

## 📖 Documentation

### Main Docs
- **[README.md](README.md)** - Complete feature documentation, troubleshooting, and architecture
- **[QUICKSTART.md](QUICKSTART.md)** - Step-by-step deployment with verification
- **[DEPENDENCIES.md](DEPENDENCIES.md)** - Detailed dependency tree and alternatives

### GitOps Deployment
- **[GitOps Guide](../../gitops/apps/SURFSENSE-GITOPS.md)** - Complete ArgoCD deployment guide
- **[Sync Waves](../../gitops/apps/SURFSENSE-SYNC-WAVES.md)** - Detailed sync wave architecture
- **[Quick Reference](../../gitops/apps/SURFSENSE-QUICK-REFERENCE.md)** - Common commands cheat sheet

### Key Sections in README
- Architecture diagram
- Prerequisites checklist
- Configuration options
- Troubleshooting guide
- Integration examples
- Browser extension setup

## 🎨 Architecture Highlights

### Data Flow
```
User → OAuth Proxy → SurfSense UI
                          ↓
                    SurfSense Backend
                          ↓
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
    PostgreSQL        Docling          RHOAI
    (pgvector)       (Parsing)      (LLM/Embed)
         ↓                                 ↓
      Redis                           SearXNG
     (Cache)                        (Web Search)
```

### Security Model
- **OpenShift OAuth** - User authentication
- **RBAC** - Cross-namespace access to RHOAI
- **SCC** - anyuid for PostgreSQL
- **Secrets** - Kubernetes secrets for credentials
- **Network** - Internal cluster communication only

## 🆚 Comparison with AnythingLLM

| Aspect | SurfSense | AnythingLLM |
|--------|-----------|-------------|
| **Architecture** | Microservices (6 containers) | Monolithic (1 container) |
| **Vector DB** | PostgreSQL + pgvector | LanceDB (built-in) |
| **Document Parsing** | Docling (external service) | Built-in |
| **Web Search** | SearXNG (optional) | Not included |
| **Browser Extension** | ✅ Included | ❌ Not available |
| **RAG Approach** | Two-tiered hybrid | Single-tier |
| **Complexity** | Higher | Lower |
| **Scalability** | Better (separate scaling) | Limited |
| **Resource Usage** | Higher (~6Gi min) | Lower (~2Gi min) |

**When to use SurfSense**:
- Need advanced RAG with two-tiered retrieval
- Want browser extension for saving pages
- Need web search capabilities
- Prefer microservices architecture
- Want to scale services independently

**When to use AnythingLLM**:
- Simpler deployment needed
- Lower resource requirements
- Don't need web search
- Prefer all-in-one solution

## ✅ What's Working

Following the anythingllm pattern, this deployment includes:

1. ✅ **Namespace isolation** - Dedicated `demo-apps` namespace
2. ✅ **OAuth integration** - OpenShift OAuth proxy sidecar
3. ✅ **RHOAI access** - Cross-namespace RBAC for InferenceServices
4. ✅ **Storage persistence** - PVCs for data and database
5. ✅ **Service discovery** - Kubernetes services for all components
6. ✅ **Security** - Proper RBAC and SCC bindings
7. ✅ **Routing** - OpenShift route with TLS edge termination
8. ✅ **Configuration** - Helm values for easy customization
9. ✅ **Documentation** - Comprehensive guides and troubleshooting

## 🔮 Optional Enhancements

Not included but easily added:

- **GPU support** - For Docling acceleration
- **Monitoring** - Prometheus metrics
- **Backup** - Velero for PostgreSQL
- **HPA** - Horizontal Pod Autoscaling
- **Multiple replicas** - For high availability
- **External services** - Use external PostgreSQL/Redis
- **Custom domains** - Custom route hostnames
- **LangSmith** - LLM observability

## 📝 Notes

### Fully Open Source
All components use open-source software:
- SurfSense (open-source)
- Docling (MIT License)
- SearXNG (AGPL-3.0)
- PostgreSQL (PostgreSQL License)
- Redis (BSD-3-Clause)

No paid API keys required for core functionality!

### Privacy-Focused
- Document parsing happens locally (Docling)
- Web search is self-hosted (SearXNG)
- All data stays in your cluster
- RHOAI models are internal

### Following Best Practices
- Based on proven anythingllm deployment pattern
- Uses OpenShift-native authentication (OAuth proxy)
- Proper RBAC and security contexts
- Health checks and probes
- Persistent storage for stateful components
- Configurable via Helm values

## 🤝 Getting Help

If you run into issues:
1. Check **[README.md](README.md)** troubleshooting section
2. Use **[QUICKSTART.md](QUICKSTART.md)** verification steps
3. Review **[DEPENDENCIES.md](DEPENDENCIES.md)** for component details
4. Check pod logs: `oc logs -n demo-apps deployment/<name>`
5. Verify RHOAI connectivity from pods

## 🎉 Ready to Deploy!

You now have a complete, production-ready SurfSense deployment that:
- Follows OpenShift/RHOAI best practices
- Uses 100% open-source components
- Includes comprehensive documentation
- Provides web search capabilities via SearXNG
- Integrates document parsing via Docling
- Connects to RHOAI for LLM and embeddings

**Next step**: Follow **[QUICKSTART.md](QUICKSTART.md)** to deploy!
