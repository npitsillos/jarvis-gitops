# Tailscale Remote Access

## Overview

Remote access to cluster services from outside the LAN is provided via Tailscale. The setup uses two parallel Gateway API implementations:

- **Cilium Gateway** (`gatewayClassName: cilium`) - handles LAN traffic via L2 announcements at VIP 192.168.10.50
- **Envoy Gateway** (`gatewayClassName: tailscale`) - handles Tailscale traffic via a Tailscale-provisioned LoadBalancer

Both gateways serve the same hostnames with the same cert-manager TLS certificates. HTTPRoutes reference both gateways via dual `parentRefs`.

## Architecture

```
LAN path:
  Browser -> DNS (192.168.10.50) -> L2/ARP -> Node -> BPF TPROXY -> Cilium Envoy -> Backend Service

Tailscale path:
  Browser -> DNS (Tailscale IP) -> WireGuard -> Tailscale Proxy Pod
    -> Envoy Gateway Pod -> Backend Service
```

## Components

### Tailscale Operator
- Installed via Helm chart `tailscale-operator` from `https://pkgs.tailscale.com/helmcharts`
- OAuth credentials stored as a SOPS-encrypted secret (`operator-oauth.yaml`)
- Runs in the `tailscale` namespace

### Envoy Gateway
- Installed via OCI Helm chart `oci://docker.io/envoyproxy/gateway-helm`
- Runs in `envoy-gateway-system` namespace
- Provides a second Gateway API implementation alongside Cilium

### EnvoyProxy Resource
Configures Envoy Gateway to use Tailscale for its LoadBalancer:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: tailscale-proxy
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        type: LoadBalancer
        loadBalancerClass: tailscale
        annotations:
          tailscale.com/hostname: jarvis-gateway
```

The `loadBalancerClass: tailscale` tells Kubernetes to use the Tailscale operator to provision the LoadBalancer. The operator creates a Tailscale device that proxies traffic to the Envoy Gateway pod.

### GatewayClass
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: tailscale
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: tailscale-proxy
    namespace: envoy-gateway-system
```

### Gateway
The tailscale gateway mirrors the Cilium gateway's listeners (same hostnames, same TLS config, same cert-manager issuer) but uses `gatewayClassName: tailscale`.

### HTTPRoutes
Each HTTPRoute has two `parentRefs` - one for each gateway:

```yaml
spec:
  parentRefs:
    - name: http-gateway        # Cilium - LAN traffic
      namespace: default
    - name: tailscale-gateway   # Envoy Gateway - Tailscale traffic
      namespace: tailscale
```

## DNS Setup

DNS must resolve `*.jarvis-home.dev` to the correct IP depending on whether the client is on the LAN or on Tailscale.

### Current Approach: Dual A Records in Cloudflare
Add two A records for each subdomain:
- `argocd.jarvis-home.dev` -> `192.168.10.50` (LAN VIP)
- `argocd.jarvis-home.dev` -> `<Tailscale IP>` (Tailscale gateway)

The client races both connections and uses whichever responds first:
- **On LAN:** LAN VIP responds, Tailscale IP unreachable -> connects via LAN
- **On Tailscale off-LAN:** Tailscale IP responds, LAN VIP unreachable -> connects via Tailscale
- **On LAN + Tailscale:** Both work, fastest wins

### Future Approach: Split DNS with Pi-hole/AdGuard
Configure Tailscale's split DNS in the admin console to forward `jarvis-home.dev` queries to a local DNS server (Pi-hole, AdGuard) that returns the Tailscale gateway IP. This avoids the connection-racing delay of dual A records.

## Hurdle 1: Why Not Just Expose the Cilium Gateway via Tailscale?

### What We Tried
Annotated the Cilium gateway service with `tailscale.com/expose: "true"` so the Tailscale operator would create a proxy for it.

### Why It Failed
The Cilium gateway service uses a **dummy endpoint** (`192.192.192.192:9999`). Traffic routing relies entirely on BPF TPROXY hooks at the network device level. The Tailscale proxy pod sends traffic to the ClusterIP, but the dummy endpoint means the traffic goes nowhere.

We also tried:
- `socketLB.enabled=true` + `socketLB.hostNamespaceOnly=false` - didn't help because socket LB still resolves to the dummy endpoint
- Direct access to Envoy on port 9964 - Envoy expects BPF-prepared traffic, not raw connections
- Annotating with `spec.infrastructure.annotations` on the Gateway resource - annotations propagated correctly, but the underlying connectivity issue remained

### Solution
Deploy Envoy Gateway as a second Gateway API implementation. Envoy Gateway creates regular Kubernetes services with real pod endpoints, so the Tailscale proxy can reach them normally.

## Hurdle 2: Tailscale Connector (Subnet Router) Can't Reach Gateway

### What We Tried
Deployed a Tailscale Connector (subnet router) advertising `192.168.10.0/24`. The idea was that remote Tailscale devices would access the gateway VIP (192.168.10.50) through the subnet router.

### Why It Failed
The subnet router pod runs inside the cluster. When it tries to forward traffic to 192.168.10.50, it hits the same pod-to-gateway limitation - the BPF TPROXY hooks don't intercept pod-originated traffic.

### Note
The Connector/subnet router is still deployed and useful for accessing other LAN devices (SSH, etc.) via Tailscale. It just can't route traffic to the Cilium gateway VIP.

## Hurdle 3: Duplicate Tailscale Device Names

### Symptom
After removing the old Tailscale gateway proxy and creating a new one via Envoy Gateway, the new device registered as `jarvis-gateway-1` instead of `jarvis-gateway`.

### Root Cause
The old device name was still registered in the Tailscale admin console even though the pod was deleted.

### Fix
1. Delete the old device from the Tailscale admin console
2. Delete the Tailscale state secret: `kubectl delete secret <ts-state-secret> -n tailscale`
3. Delete the pod to force re-registration: `kubectl delete pod <pod-name> -n tailscale`

The new pod will register with the correct hostname.

## Hurdle 4: Stale Tailscale Annotations on Cilium Gateway

### Symptom
After removing `spec.infrastructure.annotations` from the Gateway YAML, the Tailscale annotations kept reappearing on the service.

### Root Cause
Cilium propagates annotations through the CiliumEnvoyConfig (CEC) resource, not just the Gateway. The CEC cached the old annotations.

### Fix
Remove annotations from both the CEC and the service:

```bash
kubectl annotate cec cilium-gateway-http-gateway -n default tailscale.com/expose- tailscale.com/hostname-
kubectl annotate svc cilium-gateway-http-gateway -n default tailscale.com/expose- tailscale.com/hostname-
```

## Tailscale IP Stability

The Tailscale IP assigned to the gateway proxy can change if:
- The state secret is deleted
- The device is removed from the admin console and re-registered

When the IP changes, update the Cloudflare A records accordingly. Consider using Tailscale's MagicDNS hostname (`jarvis-gateway.<tailnet>.ts.net`) as a more stable reference when setting up split DNS in the future.
