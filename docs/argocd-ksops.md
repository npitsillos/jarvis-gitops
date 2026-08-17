# ArgoCD and KSOPS

## Overview

ArgoCD manages all cluster resources via GitOps. Secrets are encrypted with SOPS (using GPG key `859403B8C0B3AD6E537C4441F4EED619DF59473B`) and decrypted at sync time by KSOPS, a kustomize plugin running in the ArgoCD repo server.

## KSOPS Setup

### How It Works
1. The ArgoCD repo server has KSOPS installed as a kustomize exec plugin
2. A `ksops-generator.yaml` in each directory lists SOPS-encrypted files
3. When kustomize runs, KSOPS decrypts the files using the GPG key mounted in the repo server
4. The decrypted secrets are applied to the cluster

### Required Components in the Repo Server
- **KSOPS binary** at `/usr/local/bin/ksops` and `/home/argocd/.config/kustomize/plugin/viaduct.ai/v1/ksops/ksops`
- **GPG key** imported into `/home/argocd/.gnupg`
- **Environment variables**: `GNUPGHOME=/home/argocd/.gnupg`, `XDG_CONFIG_HOME=/home/argocd/.config`

### Repo Server Patch (argocd-repo-server-patch.yaml)
```yaml
initContainers:
  - name: install-ksops
    image: viaductoss/ksops:v4.5.1
    command: ["/usr/local/bin/ksops", "install", "/custom-tools"]
    volumeMounts:
      - mountPath: /custom-tools
        name: custom-tools
  - name: import-gpg-key
    image: quay.io/argoproj/argocd:latest
    command: ["gpg", "--batch", "--import", "/sops-gpg/sops.asc"]
    env:
      - name: GNUPGHOME
        value: /gnupg-home/.gnupg
    volumeMounts:
      - mountPath: /sops-gpg
        name: sops-gpg
      - mountPath: /gnupg-home
        name: gnupg-home
containers:
  - name: argocd-repo-server
    volumeMounts:
      - mountPath: /usr/local/bin/ksops
        name: custom-tools
        subPath: ksops
      - mountPath: /home/argocd/.config/kustomize/plugin/viaduct.ai/v1/ksops/ksops
        name: custom-tools
        subPath: ksops
      - mountPath: /home/argocd/.gnupg
        name: gnupg-home
        subPath: .gnupg
```

### KSOPS Generator Pattern
Each directory with encrypted secrets needs a `ksops-generator.yaml`:

```yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-name
  annotations:
    config.kubernetes.io/function: |
        exec:
          path: ksops
files:
  - ./encrypted-secret.yaml
```

And a reference in `kustomization.yaml`:

```yaml
generators:
  - ksops-generator.yaml
```

## Hurdle 1: Kustomize v5.3.0 / Helm v4 Incompatibility

### Symptom
ArgoCD failed to render Helm charts in kustomize with an error about `helm version -c` flag not being recognized.

### Root Cause
KSOPS v4.5.1 bundles kustomize v5.3.0, which calls `helm version -c -short` to detect Helm. Helm v4 removed the `-c` flag (it was deprecated in Helm v3).

### Fix
Do NOT use `--with-kustomize` when installing KSOPS. This way, the ArgoCD repo server uses its built-in kustomize (v5.8.1) which is compatible with Helm v4.

```yaml
# Correct - uses ArgoCD's built-in kustomize
command: ["/usr/local/bin/ksops", "install", "/custom-tools"]

# Wrong - installs kustomize v5.3.0 which breaks Helm v4
command: ["/usr/local/bin/ksops", "install", "--with-kustomize", "/custom-tools"]
```

Also do NOT mount kustomize from the custom-tools volume. Remove any volume mount for `/usr/local/bin/kustomize`.

### Version Sensitivity
This is a version-sensitive issue. When upgrading KSOPS, check which version of kustomize it bundles. When upgrading Helm, check for removed flags. When upgrading ArgoCD, check the built-in kustomize version.

## Hurdle 2: ArgoCD Pods Deleted by Prune

### Symptom
All ArgoCD pods were deleted during a resync.

### Root Cause
The `argocd-cm-extension` ArgoCD Application had `prune: true` in its sync policy. When ArgoCD detected drift in its own namespace, it pruned resources it didn't recognize - including its own pods.

### Fix
Reinstall ArgoCD manually:

```bash
kubectl apply -k /path/to/argocd/ --server-side --force-conflicts
```

### Lesson
Be very careful with `prune: true` on applications that manage ArgoCD itself. Consider using `prune: false` for the ArgoCD self-management application, or use sync waves to ensure ArgoCD components are never pruned.

## Hurdle 3: ArgoCD Password Lost After Reinstall

### Symptom
After reinstalling ArgoCD, the admin password was reset and the previous password no longer worked.

### Root Cause
The `argocd-secret` was recreated with a new random password.

### Fix
Generate a bcrypt hash of the desired password and patch the secret:

```bash
# Generate bcrypt hash
htpasswd -nbBC 10 "" 'your-password' | tr -d ':\n' | sed 's/$2y/$2a/'

# Patch the secret
kubectl -n argocd patch secret argocd-secret -p \
  '{"stringData": {"admin.password": "<bcrypt-hash>", "admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'
```

## Hurdle 4: Port Forwarding Doesn't Work for Gateway Services

### Symptom
`kubectl port-forward` to the Cilium gateway service didn't work.

### Root Cause
The Cilium gateway service has a dummy endpoint (`192.192.192.192:9999`). Port forwarding requires real endpoints to forward to.

### Workaround
Port-forward directly to the ArgoCD server pod or service instead:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Encrypting Secrets with SOPS

### Prerequisites
- `sops` CLI installed
- GPG key `859403B8C0B3AD6E537C4441F4EED619DF59473B` available in your keyring

### Encrypting a Secret
```bash
sops --encrypt \
  --pgp 859403B8C0B3AD6E537C4441F4EED619DF59473B \
  --encrypted-regex '^(data|stringData)$' \
  secret.yaml > secret.enc.yaml
```

### Decrypting (for inspection)
```bash
sops --decrypt secret.enc.yaml
```
