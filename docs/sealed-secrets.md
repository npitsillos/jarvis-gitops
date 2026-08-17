# Sealed Secrets

## Overview

Bitnami Sealed Secrets is deployed for encrypting Kubernetes secrets that can be safely stored in git. It runs in the `kube-system` namespace, installed via Helm through a kustomize `helmCharts` block.

## Hurdle 1: Helm Repository URL Changed

### Symptom
ArgoCD returned a 404 error when trying to fetch the Sealed Secrets Helm chart.

### Root Cause
The Sealed Secrets Helm repository URL changed from the old location to a new one:

- **Old (broken):** `https://bitnami-labs.github.io/sealed-secrets`
- **New (working):** `https://bitnami.github.io/sealed-secrets`

### Fix
Update `bitnami-sealed-secrets/kustomization.yaml`:

```yaml
helmCharts:
  - name: sealed-secrets
    repo: https://bitnami.github.io/sealed-secrets  # NOT bitnami-labs
    version: 2.18.0
    releaseName: sealed-secrets
    namespace: kube-system
```

### Lesson
Helm repository URLs can change without warning. When a chart fetch fails with 404, always check if the repository has been moved or renamed. Pin the chart version to avoid surprise breaks.

## Hurdle 2: Sealed Secrets CRD Not Installed

### Symptom
Resources that depend on the `SealedSecret` CRD (e.g., Longhorn's sealed secrets) failed with "no matches for kind SealedSecret".

### Root Cause
Kustomize's `helmCharts` rendering does **not** install Helm CRDs. Unlike `helm install` which processes the `crds/` directory, kustomize only templates the chart and does not handle CRD lifecycle.

### Fix
Manually install the CRD with server-side apply:

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/bitnami-labs/sealed-secrets/main/helm/sealed-secrets/crds/bitnami.com_sealedsecrets.yaml
```

### Lesson
When using kustomize `helmCharts` (which ArgoCD uses), CRDs must be installed separately. Options:
1. Apply CRDs manually before the Helm chart syncs
2. Create a separate ArgoCD Application for CRDs at an earlier sync wave
3. Use `valuesInline: { installCRDs: true }` if the chart supports it (Sealed Secrets does, and this is already set)

Note: Even with `installCRDs: true`, kustomize rendering may not include CRDs depending on the chart structure. Always verify CRDs exist after first sync.
