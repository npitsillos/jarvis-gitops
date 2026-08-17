# General Kubernetes Pitfalls

## CRD Annotation Size Limit (262KB)

### Symptom
ArgoCD sync fails with:
```
CustomResourceDefinition is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

### Root Cause
By default, `kubectl apply` stores the entire resource manifest in the `kubectl.kubernetes.io/last-applied-configuration` annotation. Large CRDs (like those from Envoy Gateway, ArgoCD's ApplicationSets, or Gateway API) exceed the 262KB annotation limit.

### Fix
Use **Server-Side Apply**, which doesn't store the last-applied-configuration annotation.

#### For ArgoCD Applications
Add `ServerSideApply=true` to the sync options:

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
```

In this repo, the `applications.yaml` template supports per-app sync options:

```yaml
# In values.yaml
- name: envoy-gateway
  syncOptions:
    - ServerSideApply=true
  ...
```

#### For Manual Applies
```bash
kubectl apply --server-side -f large-crd.yaml
```

#### Fixing Existing CRDs
If a CRD already has an oversized annotation, remove it first:

```bash
kubectl annotate crd <crd-name> kubectl.kubernetes.io/last-applied-configuration-
```

### Affected Components in This Cluster
- **Envoy Gateway CRDs**: `envoyproxies.gateway.envoyproxy.io`, `securitypolicies.gateway.envoyproxy.io`
- **Gateway API CRDs**: `httproutes.gateway.networking.k8s.io`
- **ArgoCD CRDs**: `applicationsets.argoproj.io`

### Note on ArgoCD + ServerSideApply
When ArgoCD starts a sync operation, it may cache the sync method. If you add `ServerSideApply=true` while a failed sync is retrying, the retries may still use client-side apply. To force a fresh sync:

1. Clear the stuck operation: `kubectl patch app <name> -n argocd --type merge -p '{"operation": null}'`
2. Or wait for all retries to exhaust, then ArgoCD will start a new sync with the updated options

In some cases (like this cluster's initial Envoy Gateway install), you may need to manually install the oversized CRDs with `kubectl apply --server-side` first, then let ArgoCD manage them going forward.

## Gateway API CRD Conflicts

### Symptom
Server-side apply fails with field manager conflicts on Gateway API CRDs.

### Root Cause
Multiple controllers (Cilium, Envoy Gateway) may try to manage the same Gateway API CRDs. Each controller ships its own version of the CRDs.

### Fix
Use `--force-conflicts` for manual applies, or let one controller own the CRDs:

```bash
kubectl apply --server-side --force-conflicts -f gateway-api-crds.yaml
```

In practice, whichever controller installs first owns the CRDs. The other controller will use them as-is. Ensure CRD versions are compatible between controllers.

### Version Tracking
When upgrading Cilium or Envoy Gateway, check if they ship different versions of Gateway API CRDs. Incompatible CRD versions can break one or both controllers.

## Kustomize helmCharts vs Helm Install

### Key Differences
| Feature | `helm install` | kustomize `helmCharts` |
|---------|---------------|----------------------|
| CRD installation | Yes (from `crds/` dir) | No |
| Hooks (pre-install, etc.) | Yes | No |
| Release tracking | Yes (Helm secrets) | No |
| Template rendering | Yes | Yes |
| Post-renderer | No | Yes (kustomize patches) |

### Implications for This Cluster
- CRDs must be installed separately when using kustomize `helmCharts`
- Helm hooks (like cert-gen jobs) may not run - check if the ArgoCD Application handles them via sync waves
- No `helm list` visibility for kustomize-rendered charts

## Storage: Mounting SSDs on Raspberry Pi

### Setup
The SSD connected to the jarvis node is mounted at `/mnt/ssd`:

```bash
# Find the device
lsblk

# Create mount point
sudo mkdir -p /mnt/ssd

# Get UUID
sudo blkid /dev/sda1

# Add to /etc/fstab for persistent mounting
UUID=<uuid> /mnt/ssd ext4 defaults,nofail 0 2

# Mount
sudo mount -a
```

### Longhorn Configuration
Longhorn needs the disk configured with an `ssd` tag pointing to `/mnt/ssd` on the jarvis node. This is done through the Longhorn UI or via a Longhorn `Node` resource.

## Version Upgrade Checklist

When upgrading any component, check these items:

### Cilium
- [ ] Check if Gateway API CRD versions changed
- [ ] Check for BPF behavior changes (especially TPROXY, socket LB)
- [ ] Check if Envoy version/config format changed
- [ ] Verify L2 announcement compatibility
- [ ] Test both LAN and Tailscale access paths after upgrade

### Envoy Gateway
- [ ] Check if CRDs changed (may need manual server-side apply)
- [ ] Check if Gateway API CRD versions conflict with Cilium
- [ ] Verify the EnvoyProxy CRD schema hasn't changed

### Tailscale Operator
- [ ] Check Connector CRD changes
- [ ] Check if `loadBalancerClass: tailscale` behavior changed
- [ ] Verify OAuth token scopes still sufficient

### ArgoCD
- [ ] Check built-in kustomize version (affects KSOPS compatibility)
- [ ] Check if Helm version changed (affects kustomize helmCharts rendering)
- [ ] Verify KSOPS plugin path hasn't changed

### KSOPS
- [ ] Check bundled kustomize version (must be compatible with ArgoCD's Helm)
- [ ] If using `--with-kustomize`, verify it works with current Helm version
- [ ] Check if the exec plugin path changed

### Helm Charts (All)
- [ ] Check if repository URLs changed (e.g., Sealed Secrets bitnami-labs -> bitnami)
- [ ] Check if chart values schema changed
- [ ] Check if CRDs are included or separate
- [ ] Read the changelog for breaking changes
