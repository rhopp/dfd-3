# DFD Deployment Guide

## Prerequisites

- OpenShift cluster with `oc` CLI configured
- `podman` or `buildah` for local image builds
- Quay.io account (or other container registry) with push access
- GCP service account credentials (JSON key file) for Vertex AI / Claude
- AWS S3 bucket with IAM user access
- KubeArchive API token

## GitOps Deployment (ArgoCD)

This is the recommended deployment method. ArgoCD manages the application
lifecycle and keeps the cluster in sync with git.

### 1. Prerequisites

- OpenShift GitOps (ArgoCD) installed on the cluster
- The ArgoCD instance has access to this git repository

### 2. Build and push the container image

```bash
podman build -f Containerfile -t quay.io/rhopp/dfd:latest .
podman push quay.io/rhopp/dfd:latest
```

### 3. Create secrets in the target namespace

Secrets are managed out-of-band (not in git). They must exist in the namespace
before ArgoCD syncs the Application. See
[manifests/base/secrets/README.md](manifests/base/secrets/README.md) for the
full list and creation commands.

```bash
oc create namespace dfd

# Database
DB_PASS=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)
oc create secret generic dfd-database -n dfd \
  --from-literal=username=dfd \
  --from-literal=password="$DB_PASS" \
  --from-literal=database=dfd \
  --from-literal=url="postgresql://dfd:${DB_PASS}@dfd-postgresql:5432/dfd"

# KubeArchive
oc create secret generic dfd-kubearchive -n dfd \
  --from-literal=base-url="$KUBEARCHIVE_URL" \
  --from-literal=token="$KUBEARCHIVE_TOKEN"

# Vertex AI
oc create secret generic dfd-vertex-ai -n dfd \
  --from-literal=project-id="$GOOGLE_CLOUD_PROJECT" \
  --from-literal=region="$GOOGLE_CLOUD_REGION"

# GCP credentials
oc create secret generic dfd-gcp-credentials -n dfd \
  --from-file=credentials.json=/path/to/your/credentials.json

# AWS credentials
oc create secret generic dfd-aws-credentials -n dfd \
  --from-literal=access-key-id="$AWS_ACCESS_KEY_ID" \
  --from-literal=secret-access-key="$AWS_SECRET_ACCESS_KEY"

# AWS S3
oc create secret generic dfd-aws-s3 -n dfd \
  --from-literal=bucket="$S3_BUCKET" \
  --from-literal=region="$AWS_REGION"

# OAuth proxy cookie
oc create secret generic dfd-oauth-proxy-cookie -n dfd \
  --from-literal=cookie-secret="$(openssl rand -base64 32)"
```

### 4. Deploy the ArgoCD Application

```bash
oc apply -f manifests/argocd/application.yaml
```

ArgoCD will automatically sync all resources (namespace, configmap, postgresql,
api, collector) from `manifests/overlays/production/`. Database migrations run
automatically via init containers on the API and Collector deployments.

### 5. Verify

```bash
# Check ArgoCD sync status
oc get application dfd -n openshift-gitops

# All pods should be Running
oc get pods -n dfd

# API route
oc get route dfd-api -n dfd -o jsonpath='https://{.spec.host}{"\n"}'

# Collector logs
oc logs deployment/dfd-collector -n dfd --tail=30
```

### Customizing for a different environment

To deploy to a different cluster or namespace:

1. Copy `manifests/overlays/production/` to a new overlay (e.g., `overlays/staging/`)
2. Edit the overlay's `kustomization.yaml`:
   - Change `namespace:` to your target namespace
   - Change `images:` to use a different tag or registry
   - Adjust `patches/configmap-values.yaml` for environment-specific settings
   - Add a `patches/storage-class.yaml` if your cluster uses a different storage class:
     ```yaml
     apiVersion: v1
     kind: PersistentVolumeClaim
     metadata:
       name: dfd-postgresql-data
     spec:
       storageClassName: your-storage-class
     ```
3. Copy `manifests/argocd/application.yaml` and update `spec.source.path` and
   `spec.destination.namespace` to point to your new overlay
4. Create the required secrets in your target namespace
5. Apply the new ArgoCD Application: `oc apply -f manifests/argocd/your-application.yaml`

## Manual Deployment (without ArgoCD)

If you prefer to deploy without GitOps, you can apply the kustomize overlay
directly:

```bash
# Create secrets first (see step 3 above)

# Apply all resources
oc apply -k manifests/overlays/production/

# Verify
oc get pods -n dfd
oc rollout status deployment/dfd-api -n dfd
oc rollout status deployment/dfd-collector -n dfd
```

Or apply individual manifests from `manifests/base/` in order:
namespace, configmap, postgresql (pvc, service, statefulset, backup-cronjob),
api (service, deployment, route), collector (deployment).

## Architecture

```
┌─────────────────────────────────────┐
│ OpenShift Namespace: dfd           │
│                                     │
│  ┌───────────┐   ┌───────────────┐  │
│  │ dfd-api  │   │ dfd-collector│  │
│  │ (uvicorn) │   │ (poller loop) │  │
│  │ :8080     │   │               │  │
│  │           │   │               │  │
│  │ + oauth   │   │               │  │
│  │   proxy   │   │               │  │
│  │   :8443   │   │               │  │
│  └─────┬─────┘   └───────┬───────┘  │
│        │                  │          │
│  ┌─────┴──────────────────┴───────┐  │
│  │     dfd-postgresql :5432      │  │
│  │     (StatefulSet + 10Gi PVC)   │  │
│  └────────────────────────────────┘  │
│                                     │
│  External: AWS S3,                   │
│            Vertex AI, KubeArchive   │
└─────────────────────────────────────┘
```

Both `dfd-api` and `dfd-collector` use the same container image
with different entry points.
