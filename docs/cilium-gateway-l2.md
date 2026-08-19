# Cilium Gateway API and L2 Announcements

## Overview

Cilium serves as both the CNI and the Gateway API implementation for LAN traffic. The gateway is exposed via L2 announcements with a dedicated LoadBalancer IP pool, allowing devices on the local network to reach services at a stable VIP (192.168.10.50).

## Helm Values (Critical Settings)

These are the complete Cilium Helm values used in this setup:

```yaml
kubeProxyReplacement: true
k8sServiceHost: 192.168.10.100
k8sServicePort: 6443
gatewayAPI:
  enabled: true
  hostNetwork:
    enabled: false
l2announcements:
  enabled: true
l7Proxy: true
bpf:
  tproxy: true          # Critical - without this, external traffic is silently dropped
devices: eth0
envoy:
  enabled: true
  securityContext:
    capabilities:
      envoy:
        - NET_ADMIN
        - SYS_ADMIN
        - NET_BIND_SERVICE    # Required for binding to port 443
      keepCapNetBindService: true
extraConfig:
  proxy-use-original-source-address: "true"
proxy:
  useOriginalSourceAddress: true
operator:
  replicas: 1
socketLB:
  enabled: false
  hostNamespaceOnly: false
```

**Note on `proxy-use-original-source-address`:** This must be `"true"` for this L2 announcement setup. Setting it to `"false"` (as suggested in [cilium/cilium#48045](https://github.com/cilium/cilium/issues/48045)) breaks same-node Envoy-to-pod connectivity — Envoy on the VIP lease holder node returns 503 for all services running on that same node, while cross-node services continue to work.

## Hurdle 1: BPF TPROXY - External Traffic Silently Dropped

### Symptom
External traffic to the gateway VIP (192.168.10.50:443) arrived at the NIC but was silently dropped. No errors in logs, no ICMP unreachable, just timeouts.

### Root Cause
Known Cilium bug (issues [#47400](https://github.com/cilium/cilium/issues/47400), [#46260](https://github.com/cilium/cilium/issues/46260), [#43819](https://github.com/cilium/cilium/issues/43819)). When Cilium uses L7 TPROXY to redirect traffic to Envoy, the kernel's `rp_filter` (reverse path filtering) drops the packet as a "martian source" because the source address doesn't match the expected interface.

### Fix
Set `bpf.tproxy=true` in the Cilium Helm values. This keeps the entire TPROXY operation within BPF, bypassing the kernel's rp_filter check.

```bash
helm upgrade cilium cilium/cilium -n kube-system --set bpf.tproxy=true
```

### Diagnostic Red Herring: `tc filter show` Showing Nothing
When debugging, `tc filter show dev eth0 ingress` showed no filters. This was a red herring - Cilium uses **TCX attach mode** which is invisible to `tc filter`. Use `bpftool net show dev eth0` instead to see BPF programs.

### Diagnostic Red Herring: `linux-modules-extra-raspi`
The package `linux-modules-extra-raspi` doesn't exist for the Raspberry Pi kernel. The TPROXY kernel modules are included in the main `linux-modules` package. This was not the root cause.

## Hurdle 2: Envoy "Cannot Bind 0.0.0.0:443: Permission Denied"

### Symptom
Envoy pods failed to start with permission denied when trying to bind to port 443.

### Root Cause
Envoy needs the `NET_BIND_SERVICE` capability to bind to privileged ports (< 1024), even without `hostNetwork`.

### Fix
Add `NET_BIND_SERVICE` to the Envoy security context capabilities in Cilium Helm values:

```yaml
envoy:
  securityContext:
    capabilities:
      envoy:
        - NET_ADMIN
        - SYS_ADMIN
        - NET_BIND_SERVICE
      keepCapNetBindService: true
```

## Hurdle 3: Gateway PROGRAMMED: False After Sync

### Symptom
After syncing the gateway manifest, the Gateway showed `PROGRAMMED: False` and the LoadBalancer service had the wrong IP.

### Root Cause
An old annotation `io.cilium/lb-ipam-ips: 100.81.60.103` (a previously hardcoded Tailscale IP) was stuck on the service, preventing the LB pool from assigning the correct IP.

### Fix
Remove the stale annotation:

```bash
kubectl annotate svc cilium-gateway-http-gateway -n default io.cilium/lb-ipam-ips-
```

**Lesson:** When changing gateway addressing, always check for leftover annotations on the generated service. Cilium propagates annotations from the Gateway's `spec.infrastructure.annotations` to the service, and old values can persist.

## Hurdle 4: 503 After Gateway Recreation

### Symptom
After deleting and recreating the gateway, all requests returned 503 Service Unavailable.

### Root Cause
Envoy's xDS configuration was not re-synced after the gateway was recreated. The Envoy proxies had stale routing configuration.

### Fix
Restart the Cilium and Cilium Envoy DaemonSets:

```bash
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout restart daemonset/cilium-envoy -n kube-system
```

Wait for all pods to stabilize before testing.

## How L2 Announcements Work

Understanding the L2 mechanism helps with debugging:

1. **CiliumLoadBalancerIPPool** defines the VIP range (e.g., 192.168.10.50/32)
2. **CiliumL2AnnouncementPolicy** selects which nodes can announce IPs
3. Cilium uses a **lease system** to assign a VIP to one node at a time
4. That node responds to **ARP requests** for the VIP on the physical interface
5. The node does NOT get the VIP as a real IP on any interface - it's purely ARP-level
6. External traffic arrives at the node's NIC destined for the VIP, and BPF intercepts it and redirects to Envoy on port 9964

**Key insight:** The VIP exists only at the ARP/L2 layer. The node never has this IP on its interface. BPF TPROXY handles the redirection from the VIP to Envoy transparently.

## Limitation: Pod-to-Gateway VIP Traffic Does Not Work

Pods inside the cluster **cannot** reach the Cilium gateway's ClusterIP or LB VIP. This is because:

- The gateway service uses a **dummy endpoint** (`192.192.192.192:9999`)
- Traffic routing relies on BPF TPROXY hooks at the network device level
- Pod-originated traffic doesn't go through these hooks
- Even with `socketLB.enabled=true` and `socketLB.hostNamespaceOnly=false`, the dummy endpoint means socket LB has nowhere real to send traffic

This is why a separate Envoy Gateway is needed for Tailscale remote access (see [tailscale-remote-access.md](tailscale-remote-access.md)).
