# SurfSense Deployment for OpenShift with RHOAI

SurfSense is an open-source AI research and knowledge management platform (an alternative to NotebookLM). This deployment includes:

- **SurfSense** - Main application with web UI and API
- **PostgreSQL with pgvector** - Vector database for embeddings and full-text search
- **Redis** - Caching and session management
- **Docling** - IBM's open-source document parsing service (PDF, DOCX, PPTX, etc.)
- **SearXNG** - Open-source metasearch engine for web search capabilities
- **RHOAI Integration** - Uses RHOAI InferenceServices for LLM and embeddings

## Quick Start (TL;DR)

1. **Find your RHOAI model endpoints:**
   ```bash
   oc get inferenceservices -A
   ```

2. **Edit the ArgoCD Application:**
   ```bash
   vi gitops/apps/surfsense.yaml
   # Update llm.baseUrl, llm.model, embedding.baseUrl, embedding.model
   ```

3. **Deploy:**
   ```bash
   oc apply -f gitops/apps/surfsense.yaml
   oc get application surfsense -n openshift-gitops -w
   ```

4. **Access:**
   ```bash
   oc get route surfsense -n demo-apps
   ```

**Important:** You MUST update the model endpoint URLs in `gitops/apps/surfsense.yaml` before deploying. The default values are placeholders and will not work.

## Architecture

```
User → OpenShift Route (HTTPS)
  → OAuth Proxy (Authentication)
    → SurfSense (Frontend + Backend + ElectricSQL)
      ↓
      ├─→ PostgreSQL + pgvector (Vector DB)
      ├─→ Redis (Cache)
      ├─→ Docling (Document Parsing)
      ├─→ SearXNG (Web Search)
      └─→ RHOAI InferenceServices (LLM + Embeddings)
```

## Prerequisites

1. **OpenShift cluster** with admin access
2. **RHOAI** deployed with at least one InferenceService for:
   - LLM (e.g., granite-7b-instruct)
   - Embeddings (can use same model or dedicated embedding model)
3. **Helm 3** installed
4. **Storage** - Dynamic provisioning or pre-created PVCs

## Deployment Options

### Option 1: GitOps Deployment with ArgoCD (Recommended)

Deploy using ArgoCD for automated GitOps workflow.

#### Step 1: Configure Model Endpoints

Before deploying, you **MUST** update the model endpoints in `gitops/apps/surfsense.yaml` to point to your RHOAI InferenceServices.

**Find your InferenceService URLs:**

```bash
# List all InferenceServices in your RHOAI namespace
oc get inferenceservices -n demo-models

# Get the internal service URL for a specific model
oc get inferenceservice granite-7b -n demo-models -o jsonpath='{.status.components.predictor.url}'

# The internal cluster URL format is:
# http://<model-name>-predictor.<namespace>.svc.cluster.local/v1
```

**Edit the ArgoCD Application:**

Edit `gitops/apps/surfsense.yaml` and update the Helm parameters with your actual endpoints:

```yaml
spec:
  source:
    helm:
      parameters:
        # Update these with your actual RHOAI InferenceService URLs
        - name: llm.baseUrl
          value: "http://granite-7b-predictor.demo-models.svc.cluster.local/v1"
        - name: llm.model
          value: "granite-7b-instruct"
        - name: llm.apiKey
          value: ""  # Leave empty for internal cluster access
        
        - name: embedding.baseUrl
          value: "http://granite-7b-predictor.demo-models.svc.cluster.local/v1"
        - name: embedding.model
          value: "granite-7b-instruct"
        - name: embedding.apiKey
          value: ""  # Leave empty for internal cluster access
```

**Common model endpoint examples:**

| Model | Typical URL | Model Name |
|-------|-------------|------------|
| Granite 7B | `http://granite-7b-predictor.demo-models.svc.cluster.local/v1` | `granite-7b-instruct` |
| Llama 3 8B | `http://llama-3-8b-predictor.demo-models.svc.cluster.local/v1` | `llama-3-8b-instruct` |
| Mistral 7B | `http://mistral-7b-predictor.demo-models.svc.cluster.local/v1` | `mistral-7b-instruct` |

> **Note:** You can use the same model for both LLM and embeddings, or use separate dedicated embedding models if available.

#### Step 2: Deploy with ArgoCD

```bash
# Apply the ArgoCD Application (after updating endpoints above)
oc apply -f gitops/apps/surfsense.yaml

# Monitor deployment progress
oc get application surfsense -n openshift-gitops -w

# Check application status
argocd app get surfsense

# View sync status and health
argocd app sync surfsense --prune
```

#### Step 3: Verify Deployment

```bash
# Check all pods are running
oc get pods -n demo-apps

# Get the route URL
oc get route surfsense -n demo-apps -o jsonpath='{.spec.host}'

# Access the application
open https://$(oc get route surfsense -n demo-apps -o jsonpath='{.spec.host}')
```

**Benefits of GitOps Deployment:**
- ✅ Version-controlled configuration
- ✅ Automated sync from Git
- ✅ Self-healing capabilities
- ✅ Proper dependency ordering
- ✅ Easy rollback
- ✅ Allows model endpoint updates without ArgoCD conflicts

**Important:** The ArgoCD Application is configured to ignore differences in model endpoints (`ignoreDifferences`), which means you can update model connections through the SurfSense UI or by updating the Application parameters without ArgoCD reverting your changes.

**How ignoreDifferences Works:**

The Application manifest includes `ignoreDifferences` configuration that tells ArgoCD to ignore changes to specific environment variables in the deployment:
- `OPENAI_API_BASE` (LLM endpoint)
- `LLM_MODEL` (LLM model name)
- `OPENAI_EMBEDDING_BASE` (Embedding endpoint)
- `EMBEDDING_MODEL` (Embedding model name)
- `OPENAI_API_KEY` (API key for external services)

This allows you to:
1. Update model endpoints via ArgoCD parameters without conflicts
2. Manually edit the deployment to test different models
3. Change endpoints through the SurfSense UI (if supported)

ArgoCD will still manage all other aspects of the deployment but won't try to revert these specific values.

### Option 2: Direct Helm Deployment

For development, testing, or manual deployment:

## Quick Start

### Prerequisites Check

Before deploying, ensure you have:

1. **RHOAI InferenceServices deployed** - At least one model serving endpoint
2. **OpenShift cluster access** - Admin or sufficient permissions to create namespaces and applications
3. **ArgoCD/OpenShift GitOps** - Installed and configured (for GitOps deployment)

### Find Your Model Endpoints

**IMPORTANT:** You need to know your RHOAI InferenceService URLs before deploying.

```bash
# List all available InferenceServices
oc get inferenceservices -A

# Example output:
# NAMESPACE      NAME          URL                                                                 READY
# demo-models    granite-7b    http://granite-7b-predictor-demo-models.apps.example.com            True

# Get the internal cluster URL (recommended for in-cluster communication)
oc get inferenceservice granite-7b -n demo-models -o jsonpath='{.status.components.predictor.url}'

# The internal URL format is:
# http://<model-name>-predictor.<namespace>.svc.cluster.local/v1
```

### Deploy with GitOps (Recommended)

See the **"Option 1: GitOps Deployment with ArgoCD"** section above for complete instructions.

Quick summary:
1. Edit `gitops/apps/surfsense.yaml` and update the `llm.baseUrl` and `embedding.baseUrl` parameters
2. Apply: `oc apply -f gitops/apps/surfsense.yaml`
3. Monitor: `oc get application surfsense -n openshift-gitops -w`

### Deploy with Helm (Development/Testing)

For manual deployment or testing:

#### Step 1: Configure Model Endpoints

You can configure endpoints in two ways:

**Option A: Edit values-rhoai.yaml (not recommended for GitOps)**

```bash
# Edit the values file
vi apps/3rd-party-apps/surfsense/helm/values-rhoai.yaml

# Update the baseUrl values:
# llm:
#   baseUrl: "http://your-model-predictor.demo-models.svc.cluster.local/v1"
```

**Option B: Override via command line (recommended)**

Pass the endpoints as Helm parameters during installation:

```bash
helm install surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  --set llm.baseUrl="http://granite-7b-predictor.demo-models.svc.cluster.local/v1" \
  --set llm.model="granite-7b-instruct" \
  --set embedding.baseUrl="http://granite-7b-predictor.demo-models.svc.cluster.local/v1" \
  --set embedding.model="granite-7b-instruct" \
  -n demo-apps --create-namespace
```

**With API keys (if needed for external endpoints):**

```bash
helm install surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  --set llm.baseUrl="https://api.openai.com/v1" \
  --set llm.model="gpt-4" \
  --set llm.apiKey="sk-your-api-key-here" \
  --set embedding.baseUrl="https://api.openai.com/v1" \
  --set embedding.model="text-embedding-3-small" \
  --set embedding.apiKey="sk-your-api-key-here" \
  -n demo-apps --create-namespace
```

#### Step 2: Access SurfSense

```bash
# Get the route URL
oc get route surfsense -n demo-apps -o jsonpath='{.spec.host}'

# Access via browser (OAuth authentication will prompt for OpenShift login)
open https://$(oc get route surfsense -n demo-apps -o jsonpath='{.spec.host}')
```

## Configuration Options

### Updating Model Endpoints After Deployment

If you need to change model endpoints after initial deployment, you have two options:

#### Option 1: Update ArgoCD Application (GitOps Method)

Edit your ArgoCD Application and update the Helm parameters:

```bash
# Edit the Application
oc edit application surfsense -n openshift-gitops

# Or edit the file and reapply
vi gitops/apps/surfsense.yaml
oc apply -f gitops/apps/surfsense.yaml

# ArgoCD will automatically sync the changes
argocd app sync surfsense
```

#### Option 2: Update via ArgoCD CLI

```bash
# Update specific parameters
argocd app set surfsense \
  --parameter llm.baseUrl="http://new-model-predictor.demo-models.svc.cluster.local/v1" \
  --parameter llm.model="new-model-name"

# Sync the changes
argocd app sync surfsense
```

#### Option 3: Direct Helm Upgrade

For non-GitOps deployments:

```bash
helm upgrade surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  --set llm.baseUrl="http://new-model-predictor.demo-models.svc.cluster.local/v1" \
  --set llm.model="new-model-name" \
  -n demo-apps
```

**Note:** The ArgoCD Application is configured with `ignoreDifferences` for model endpoints, so you can update these values without conflicts. However, for true GitOps, it's best to update the Application manifest in Git (Option 1).

### Authentication

**Local Authentication (default):**
```yaml
auth:
  type: "LOCAL"
  registrationEnabled: true
```

**Google OAuth:**
```yaml
auth:
  type: "GOOGLE"
  googleClientId: "your-client-id"
  googleClientSecret: "your-client-secret"
```

### Web Search Options

**SearXNG (default - open-source, self-hosted):**
```yaml
webSearch:
  enabled: true
  provider: "searxng"
```

**Tavily (requires API key):**
```yaml
webSearch:
  enabled: true
  provider: "tavily"
  apiKey: "your-tavily-api-key"
```

**Linkup (requires API key):**
```yaml
webSearch:
  enabled: true
  provider: "linkup"
  apiKey: "your-linkup-api-key"
```

**Disable web search:**
```yaml
webSearch:
  enabled: false
```

### Document Processing

Docling is enabled by default for privacy-focused local document processing:

```yaml
docling:
  enabled: true
etl:
  service: "DOCLING"
```

Alternative options (require API keys):
- `UNSTRUCTURED` - Cloud-based via Unstructured Platform
- `LLAMACLOUD` - LlamaIndex cloud service

### Optional Services

**Disable specific services:**

```yaml
postgresql:
  enabled: false  # Use external PostgreSQL

redis:
  enabled: false  # Use external Redis

docling:
  enabled: false  # Use external Docling or different ETL

searxng:
  enabled: false  # Disable web search
```

### Resource Configuration

Adjust resources based on your workload:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"

postgresql:
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
  storage:
    size: 20Gi

docling:
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    # Optional: Add GPU support for faster processing
    # nvidia.com/gpu: "1"
```

## Deployment Components

### Core Services

| Service | Purpose | Port | Storage |
|---------|---------|------|---------|
| SurfSense | Main application | 3000 (frontend), 8000 (backend), 5133 (electric) | 10Gi PVC |
| PostgreSQL | Vector DB with pgvector | 5432 | 20Gi PVC |
| Redis | Cache & sessions | 6379 | None (ephemeral) |
| Docling | Document parsing | 5001 | None |
| SearXNG | Web search | 8080 | None |
| OAuth Proxy | Authentication | 8443 | None |

### RBAC Permissions

The deployment creates:
- **ServiceAccount**: `surfsense` in `demo-apps` namespace
- **RoleBinding**: Grants `anyuid` SCC for PostgreSQL
- **Cross-namespace RoleBinding**: Grants access to RHOAI InferenceServices in `demo` namespace

## Troubleshooting

### Check Pod Status

```bash
# View all pods
oc get pods -n demo-apps

# Check SurfSense logs
oc logs -n demo-apps deployment/surfsense -c surfsense --tail=100

# Check PostgreSQL logs
oc logs -n demo-apps deployment/postgresql --tail=100

# Check Docling logs
oc logs -n demo-apps deployment/docling --tail=100
```

### Common Issues

**PostgreSQL not ready:**
```bash
# Check if pgvector extension is loaded
oc exec -n demo-apps deployment/postgresql -- psql -U surfsense -d surfsense -c "SELECT * FROM pg_extension WHERE extname = 'vector';"
```

**Database connection issues:**
```bash
# Verify database URL in pod
oc exec -n demo-apps deployment/surfsense -c surfsense -- env | grep DATABASE_URL
```

**RHOAI connection issues:**
```bash
# Test connection to InferenceService from SurfSense pod
oc exec -n demo-apps deployment/surfsense -c surfsense -- curl -v http://granite-7b-predictor.demo.svc.cluster.local/v1/models
```

**Docling not parsing documents:**
```bash
# Check Docling service health
oc exec -n demo-apps deployment/surfsense -c surfsense -- curl -v http://docling-service:5001/docs
```

### Verify pgvector Installation

If you need to manually create the pgvector extension:

```bash
oc exec -n demo-apps deployment/postgresql -- psql -U surfsense -d surfsense -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

## Upgrading

```bash
# Upgrade with new values
helm upgrade surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  -n demo-apps

# Rollback if needed
helm rollback surfsense -n demo-apps
```

## Uninstalling

```bash
# Delete Helm release (keeps PVCs by default)
helm uninstall surfsense -n demo-apps

# Delete PVCs if needed
oc delete pvc -n demo-apps -l app=surfsense
oc delete pvc -n demo-apps -l app=postgresql
```

## Features

- **RAG (Retrieval-Augmented Generation)** - Two-tiered RAG with document and chunk-level retrieval
- **Hybrid Search** - Combines semantic (vector) and full-text (PostgreSQL tsquery) search
- **Document Parsing** - Supports PDF, DOCX, PPTX, HTML, images, CSV via Docling
- **Web Search** - Autonomous web search using SearXNG
- **Browser Extension** - Chrome/Firefox extension for saving web pages to knowledge base
- **Multi-LLM Support** - Compatible with any OpenAI-compatible API (RHOAI, Ollama, etc.)
- **Privacy-Focused** - All processing happens locally or in your cluster

## Browser Extension

SurfSense includes browser extensions for Chrome and Firefox that allow you to:
- Save web pages directly to your knowledge base
- Capture content from authenticated pages
- Organize and tag saved content

**Installation:**
1. Access SurfSense UI via the OpenShift route
2. Navigate to Settings → Browser Extension
3. Follow instructions to install for your browser

## Integration with RHOAI Models

### Recommended Models

**LLM:**
- granite-7b-instruct
- llama-3-8b-instruct
- mistral-7b-instruct

**Embeddings:**
- granite-embedding (if available)
- Use same LLM model for embeddings (works but less optimal)
- sentence-transformers/all-MiniLM-L6-v2 (local, built-in option)

### Testing RHOAI Integration

After deployment, verify that SurfSense can connect to your RHOAI models:

#### Test from SurfSense Pod

```bash
# Get a shell in the SurfSense pod
oc rsh -n demo-apps deployment/surfsense -c surfsense

# Test LLM endpoint
curl -X POST http://granite-7b-predictor.demo-models.svc.cluster.local/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-7b-instruct",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'

# Test embedding endpoint
curl -X POST http://granite-7b-predictor.demo-models.svc.cluster.local/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "granite-7b-instruct",
    "input": "Test embedding"
  }'
```

#### Check Environment Variables

Verify the correct endpoints are configured:

```bash
# Check LLM configuration
oc exec -n demo-apps deployment/surfsense -c surfsense -- env | grep -E "OPENAI_API_BASE|LLM_MODEL|EMBEDDING"

# Expected output:
# OPENAI_API_BASE=http://granite-7b-predictor.demo-models.svc.cluster.local/v1
# LLM_MODEL=granite-7b-instruct
# OPENAI_EMBEDDING_BASE=http://granite-7b-predictor.demo-models.svc.cluster.local/v1
# EMBEDDING_MODEL=granite-7b-instruct
```

#### Test from SurfSense UI

1. Access SurfSense via the route
2. Create a new project
3. Upload a document (PDF, DOCX, etc.)
4. Ask a question about the document
5. Check that responses are generated successfully

### Recommended Models

**LLM Options:**
- **Granite 7B Instruct** - Good balance of performance and resource usage
- **Llama 3 8B Instruct** - Strong general-purpose model
- **Mistral 7B Instruct** - Fast inference, good quality

**Embedding Options:**
- **Same as LLM** - Use the same model for both (simpler, but less optimal)
- **Dedicated embedding model** - Better quality if you have a separate embedding model deployed
- **Built-in sentence-transformers** - Fallback option, runs locally in SurfSense pod

## Support

For issues with:
- **SurfSense**: https://github.com/MODSetter/SurfSense
- **Docling**: https://github.com/docling-project/docling
- **SearXNG**: https://github.com/searxng/searxng
- **RHOAI**: https://access.redhat.com/products/red-hat-openshift-ai

## License

- SurfSense: Check upstream repository
- Docling: MIT License
- SearXNG: AGPL-3.0
- PostgreSQL: PostgreSQL License
- Redis: BSD-3-Clause
