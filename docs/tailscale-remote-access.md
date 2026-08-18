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

### LAN Clients
Cloudflare DNS has a wildcard A record for `*.jarvis-home.dev` pointing to the Cilium VIP (`192.168.10.50`). LAN devices use their default DNS (via the router) which resolves through Cloudflare.

### Tailscale Clients: Split DNS with Technitium
Technitium DNS Server runs in the cluster as a Helm chart (`technitium/` directory) and provides split-horizon DNS using the Split Horizon app.

**Tailscale admin console configuration:**
- Split DNS domain: `jarvis-home.dev`
- Nameserver: `192.168.10.100` (Technitium DNS via `externalIPs` on the `jarvis` node)

**Technitium Split Horizon APP record** for `*.jarvis-home.dev`:
```json
{
  "192.168.10.0/24": ["192.168.10.50"],
  "100.64.0.0/10": ["100.122.181.96"],
  "10.0.0.0/16": ["100.122.181.96"]
}
```

- `192.168.10.0/24` (LAN) → Cilium VIP
- `100.64.0.0/10` (Tailscale CGNAT range) → Tailscale gateway
- `10.0.0.0/16` (pod network) → Tailscale gateway (needed because the subnet router pod proxies DNS queries from its pod IP, not the original Tailscale client IP)

**Traffic flow:**
```
Phone (mobile data) -> Tailscale MagicDNS
  -> Split DNS forwards jarvis-home.dev to 192.168.10.100
  -> Subnet router routes to Technitium pod
  -> Split Horizon returns 100.122.181.96 (Tailscale gateway)
  -> Phone connects to 100.122.181.96:443
  -> Tailscale gateway (Envoy) -> HTTPRoute -> Backend service
```

### Why `externalIPs` is Required on the Technitium DNS Service

The Technitium DNS LoadBalancer service gets a Cilium VIP (`192.168.10.51`), but the Tailscale subnet router pod cannot reach Cilium VIPs (see [cilium-gateway-l2.md](cilium-gateway-l2.md) - "Limitation: Pod-to-Gateway VIP Traffic Does Not Work"). The `externalIPs: ["192.168.10.100"]` binds port 53 directly on the `jarvis` node's real IP, which the subnet router pod can reach. A `nodeSelector` pins Technitium to the `jarvis` node to ensure the externalIP remains valid.

### Why the Tailscale Operator ProxyClass Cannot Fix This

The Tailscale operator's `ProxyClass` CRD does not expose a `hostNetwork` field for pod customization. If it did, the subnet router pod could run on the host network and reach Cilium VIPs directly, eliminating the need for `externalIPs`.

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
