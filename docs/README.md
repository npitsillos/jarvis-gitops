# Jarvis GitOps Documentation

This directory documents the hurdles, decisions, and solutions encountered while setting up the Jarvis K3s homelab cluster. These notes serve as a reference for replicating or troubleshooting the setup in the future.

## Architecture Overview

- **K3s v1.36.3** on Raspberry Pi (arm64, Ubuntu 26.04, kernel 7.0.0)
- **Cilium** as CNI with kube-proxy replacement, Gateway API, and L2 announcements
- **Envoy Gateway** as a second Gateway API implementation for Tailscale remote access
- **Tailscale Operator** for remote access via the tailnet
- **ArgoCD** with KSOPS for GitOps and secret management
- **cert-manager** with Let's Encrypt for TLS certificates
- **Longhorn** for persistent storage (SSD-backed)
- **Sealed Secrets** for encrypting secrets in git

## Documents

- [Cilium Gateway API and L2 Announcements](cilium-gateway-l2.md) - BPF TPROXY, Envoy permissions, L2 networking
- [Tailscale Remote Access](tailscale-remote-access.md) - Subnet router, Envoy Gateway, split DNS
- [ArgoCD and KSOPS](argocd-ksops.md) - KSOPS setup, kustomize/Helm compatibility, CRD issues
- [Sealed Secrets](sealed-secrets.md) - Repo URL migration, CRD installation
- [General Kubernetes Pitfalls](general-k8s-pitfalls.md) - CRD annotation limits, server-side apply, version tracking

## Important: Version Tracking

When upgrading any component, always check changelogs and migration guides. Several issues in this setup were caused by breaking changes between versions:

- **Helm v3 to v4**: Removed `helm version -c` flag, breaking kustomize v5.3.0
- **KSOPS versions**: Bundled kustomize version may lag behind, causing incompatibilities
- **Cilium versions**: BPF behavior and Gateway API support evolve significantly between releases
- **Gateway API CRD versions**: Envoy Gateway and Cilium may ship different versions of the same CRDs
- **Sealed Secrets**: Helm repo URL changed from `bitnami-labs` to `bitnami`

Always pin versions in Helm charts and gitops manifests, and test upgrades in isolation before committing.
