# SurfSense Quick Deployment Guide

## Prerequisites Checklist

- [ ] OpenShift cluster with admin access
- [ ] RHOAI deployed with InferenceServices
- [ ] Helm 3 installed
- [ ] `oc` CLI configured and logged in

## Step-by-Step Deployment

### 1. Find your RHOAI InferenceService URLs

```bash
# List available InferenceServices
oc get inferenceservices -n demo

# Get the URL for your LLM model
oc get inferenceservice granite-7b -n demo -o jsonpath='{.status.components.predictor.url}'

# Get the URL for your embedding model (if separate)
oc get inferenceservice granite-embedding -n demo -o jsonpath='{.status.components.predictor.url}'
```

Note: The internal service URL format is typically:
`http://<model-name>-predictor.<namespace>.svc.cluster.local/v1`

### 2. Update values-rhoai.yaml

Edit `helm/values-rhoai.yaml` and replace the placeholder URLs:

```yaml
llm:
  baseUrl: "http://granite-7b-predictor.demo.svc.cluster.local/v1"
  model: "granite-7b-instruct"

embedding:
  baseUrl: "http://granite-embedding-predictor.demo.svc.cluster.local/v1"
  model: "granite-embedding"
```

### 3. Deploy with Helm

```bash
cd apps/3rd-party-apps/surfsense

# Install SurfSense
helm install surfsense ./helm \
  -f ./helm/values-rhoai.yaml \
  -n demo-apps --create-namespace

# Watch the deployment
watch oc get pods -n demo-apps
```

### 4. Wait for all pods to be ready

Expected pods:
- `surfsense-*` - Main application (2 containers: oauth-proxy + surfsense)
- `postgresql-*` - Database with pgvector
- `redis-*` - Cache
- `docling-*` - Document parser
- `searxng-*` - Web search

All should show `Running` and `Ready 1/1` (or `2/2` for surfsense).

### 5. Access SurfSense

```bash
# Get the route URL
ROUTE_URL=$(oc get route surfsense -n demo-apps -o jsonpath='{.spec.host}')
echo "Access SurfSense at: https://$ROUTE_URL"

# Open in browser
open https://$ROUTE_URL
```

You'll be prompted to log in with OpenShift OAuth.

## Verification Steps

### Check pod status
```bash
oc get pods -n demo-apps
```

### Check logs
```bash
# SurfSense application logs
oc logs -n demo-apps deployment/surfsense -c surfsense --tail=50

# PostgreSQL logs
oc logs -n demo-apps deployment/postgresql --tail=20

# Docling logs
oc logs -n demo-apps deployment/docling --tail=20
```

### Test database connection
```bash
# Exec into SurfSense pod
oc exec -n demo-apps deployment/surfsense -c surfsense -it -- /bin/sh

# Inside the pod, test database
nc -zv postgresql-service 5432

# Test Redis
nc -zv redis-service 6379

# Test Docling
curl -v http://docling-service:5001/docs

# Exit the pod
exit
```

### Test RHOAI connection
```bash
# From SurfSense pod
oc exec -n demo-apps deployment/surfsense -c surfsense -- \
  curl -s http://granite-7b-predictor.demo.svc.cluster.local/v1/models
```

## Troubleshooting

### Pod stuck in Pending
```bash
# Check events
oc describe pod <pod-name> -n demo-apps

# Common issues:
# - PVC not bound (check: oc get pvc -n demo-apps)
# - Insufficient resources (check node capacity)
```

### PostgreSQL pod CrashLoopBackOff
```bash
# Check if using correct storage class
oc get pvc postgresql-storage -n demo-apps

# Check PostgreSQL logs
oc logs -n demo-apps deployment/postgresql --tail=100
```

### SurfSense can't connect to RHOAI
```bash
# Verify cross-namespace RBAC
oc get rolebinding surfsense-inferenceservice-access -n demo

# Test connection from SurfSense pod
oc exec -n demo-apps deployment/surfsense -c surfsense -- \
  curl -v http://granite-7b-predictor.demo.svc.cluster.local/v1/models
```

### Docling not parsing documents
```bash
# Check Docling pod status
oc get pods -n demo-apps -l app=docling

# Check Docling logs
oc logs -n demo-apps deployment/docling --tail=50

# Test Docling endpoint
oc exec -n demo-apps deployment/surfsense -c surfsense -- \
  curl -v http://docling-service:5001/docs
```

## Next Steps

1. **Upload a document** - Test document parsing with Docling
2. **Try web search** - Ask a question that requires web search
3. **Install browser extension** - Navigate to Settings → Browser Extension
4. **Configure additional LLM models** - Add more RHOAI models if available

## Uninstall

```bash
# Remove the Helm release
helm uninstall surfsense -n demo-apps

# Optionally delete the namespace and all resources
oc delete namespace demo-apps

# Or keep namespace but delete PVCs
oc delete pvc -n demo-apps --all
```

## Customization

### Change authentication to Google OAuth

Edit `values-rhoai.yaml`:
```yaml
auth:
  type: "GOOGLE"
  googleClientId: "your-client-id.apps.googleusercontent.com"
  googleClientSecret: "your-secret"
```

### Disable web search

Edit `values-rhoai.yaml`:
```yaml
webSearch:
  enabled: false

searxng:
  enabled: false
```

### Adjust resources

Edit `values-rhoai.yaml`:
```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

### Use external PostgreSQL or Redis

Edit `values-rhoai.yaml`:
```yaml
postgresql:
  enabled: false

redis:
  enabled: false

# Then manually configure DATABASE_URL and REDIS_URL in the secret
```

## Support

- SurfSense: https://github.com/MODSetter/SurfSense
- Documentation: See README.md for detailed information
