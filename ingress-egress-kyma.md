# Kubernetes Network Policies for SAP BTP Kyma Runtime

A comprehensive documentation guide for configuring Ingress (incoming) and Egress (outgoing) network policies in SAP BTP Kyma clusters using Istio service mesh.

---

## Table of Contents

1. [Overview: Default Behavior vs. Secured Configuration](#1-overview-default-behavior-vs-secured-configuration)
2. [Understanding Traffic Flow](#2-understanding-traffic-flow)
3. [Ingress Policies](#3-ingress-policies-restricting-incoming-traffic)
4. [Egress Policies](#4-egress-policies-restricting-outgoing-traffic)
   - [4.7 ServiceEntry with IP Addresses](#47-serviceentry-with-ip-addresses)
   - [4.8 Kubernetes NetworkPolicy vs. Istio Policies](#48-kubernetes-networkpolicy-vs-istio-policies)
5. [ExternalTrafficPolicy Deep Dive](#5-externaltrafficpolicy-deep-dive)
6. [NATS Messaging in Kyma](#6-nats-messaging-in-kyma)
7. [AWS Load Balancer Considerations](#7-aws-load-balancer-considerations)
8. [Complete Example Scenarios](#8-complete-example-scenarios)
9. [Quick Reference Commands](#9-quick-reference-commands)

---

## 1. Overview: Default Behavior vs. Secured Configuration

### Default Kyma Behavior (Before Any Changes)

| Traffic Type | Default Behavior | Security Risk |
|--------------|------------------|---------------|
| **Ingress** | All IPs allowed access | Anyone on the internet can reach your services |
| **Egress** | All external URLs accessible | Pods can exfiltrate data to any destination |
| **Internal** | All pods can communicate | Lateral movement possible if one pod is compromised |

### Secured Configuration (After Changes)

| Traffic Type | Secured Behavior | Benefit |
|--------------|------------------|---------|
| **Ingress** | Only whitelisted IPs allowed | Only trusted sources (office, VPN) can access |
| **Egress** | Only registered external services allowed | Prevents data exfiltration to unknown destinations |
| **Internal** | Namespace/pod scoped access | Blast radius limited if one pod is compromised |

> [!IMPORTANT]
> **Why change from defaults?** In a production environment, you want to follow the **principle of least privilege**: only allow traffic that is explicitly required, and deny everything else.

---

## 2. Understanding Traffic Flow

### 2.1 Ingress Flow (External → Cluster)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              INGRESS TRAFFIC FLOW                                 │
└──────────────────────────────────────────────────────────────────────────────────┘

  User/Client                 AWS NLB                Istio Ingress           Your Pod
  (163.223.x.x)               (Public IP)            Gateway                 (garden ns)
      │                           │                      │                       │
      │  HTTPS Request            │                      │                       │
      ├──────────────────────────►│                      │                       │
      │                           │                      │                       │
      │                           │  Forward (preserves  │                       │
      │                           │  client IP if Local) │                       │
      │                           ├─────────────────────►│                       │
      │                           │                      │                       │
      │                           │                      │ AuthorizationPolicy   │
      │                           │                      │ checks remoteIpBlocks │
      │                           │                      │                       │
      │                           │                      │ ✅ IP in whitelist    │
      │                           │                      ├──────────────────────►│
      │                           │                      │                       │
      │                           │                      │ ❌ IP not in list     │
      │                           │                      │◄─── 403 Forbidden ───│
      │                           │                      │                       │
```

### 2.2 Egress Flow (Cluster → External)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              EGRESS TRAFFIC FLOW                                  │
└──────────────────────────────────────────────────────────────────────────────────┘

  Your Pod                  Envoy Sidecar            External Service
  (beautiful-garden)        (istio-proxy)            (zenquotes.io)
      │                           │                       │
      │  curl zenquotes.io        │                       │
      ├──────────────────────────►│                       │
      │                           │                       │
      │                           │ Checks Sidecar config:│
      │                           │ outboundTrafficPolicy │
      │                           │ = REGISTRY_ONLY       │
      │                           │                       │
      │                           │ Is zenquotes.io in    │
      │                           │ ServiceEntry registry?│
      │                           │                       │
      │                           │ ✅ YES - Forward      │
      │                           ├──────────────────────►│
      │                           │                       │
      │  curl google.com          │                       │
      ├──────────────────────────►│                       │
      │                           │                       │
      │                           │ ❌ NO - Block         │
      │◄──── 502 Bad Gateway ────│                       │
      │                           │                       │
```

### 2.3 Internal Traffic Flow (Pod → Pod)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           INTERNAL TRAFFIC FLOW (mTLS)                            │
└──────────────────────────────────────────────────────────────────────────────────┘

  Pod A (garden)            Envoy A              Envoy B           Pod B (garden)
  (beautiful-garden)        Sidecar              Sidecar           (jaeger-query)
      │                        │                    │                    │
      │  HTTP to jaeger:16686  │                    │                    │
      ├───────────────────────►│                    │                    │
      │                        │                    │                    │
      │                        │ TLS Handshake      │                    │
      │                        │ (mutual cert)      │                    │
      │                        ├───────────────────►│                    │
      │                        │                    │                    │
      │                        │                    │  Decrypt & Forward │
      │                        │                    ├───────────────────►│
      │                        │                    │                    │
```

---

## 3. Ingress Policies: Restricting Incoming Traffic

### 3.1 Three Levels of Ingress Control

| Level | Mechanism | Namespace | Selector | Use Case |
|-------|-----------|-----------|----------|----------|
| **Cluster-wide** | AuthorizationPolicy on `istio-ingressgateway` | `istio-system` | `app: istio-ingressgateway` | Block all external traffic except whitelisted IPs |
| **Namespace-level** | AuthorizationPolicy without workloadSelector | App namespace | None | Restrict access to specific namespace |
| **Pod-level** | AuthorizationPolicy with workloadSelector | App namespace | Pod labels | Restrict access to specific pods |

### 3.2 Cluster-Wide Ingress Policy (Recommended)

This policy restricts ALL external traffic entering the cluster to only whitelisted IPs.

```yaml
# ingress/istio-gateway-policy.yaml
# Apply: kubectl apply -f ingress/istio-gateway-policy.yaml
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: gateway-allow-specific-ips
  namespace: istio-system          # Must be in istio-system for cluster-wide
  labels:
    managed-by: network-policies
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway    # Target the ingress gateway
  action: ALLOW
  rules:
    - from:
        - source:
            remoteIpBlocks:
              # Office IPs
              - "163.223.48.163/32"
              - "103.99.8.235/32"
              # Home IPs
              - "223.196.173.216/32"
              - "223.196.169.244/32"
              # VPN Range
              - "10.0.0.0/8"
```

> [!CAUTION]
> When you create an `ALLOW` AuthorizationPolicy without a catch-all rule, Istio implicitly denies all other traffic. This means if your IP is not in the list, you will get `403 Forbidden`.

### 3.3 Namespace-Level Ingress Policy

Restricts traffic to all pods in a specific namespace based on headers/principals.

```yaml
# ingress/namespace-policy.yaml
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: garden-namespace-policy
  namespace: garden                # Applied to garden namespace
spec:
  # No selector = applies to all pods in namespace
  action: ALLOW
  rules:
    - from:
        - source:
            # Only allow traffic from istio-ingressgateway
            principals:
              - "cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
        - source:
            # Allow internal mesh traffic
            namespaces:
              - "garden"
              - "istio-system"
```

### 3.4 Pod-Level Ingress Policy

Restricts traffic to specific pods matching labels.

```yaml
# ingress/pod-policy.yaml
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: beautiful-garden-policy
  namespace: garden
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: beautiful-garden    # Only affects this pod
  action: ALLOW
  rules:
    - from:
        - source:
            remoteIpBlocks:
              - "163.223.48.163/32"
    - to:
        - operation:
            paths:
              - "/api/*"           # Only allow access to /api/* routes
              - "/public/*"
```

### 3.5 What Changes from Default (Ingress)

| Aspect | Default | After Policy | Why Change |
|--------|---------|--------------|------------|
| External access | All IPs allowed | Only whitelisted IPs | Prevent unauthorized access |
| 403 response | Never happens | Happens for blocked IPs | Security - deny unknown sources |
| IP preservation | May use SNAT | Must use `externalTrafficPolicy: Local` | Policy needs real client IP |

---

## 4. Egress Policies: Restricting Outgoing Traffic

### 4.1 Two Components Required

1. **Sidecar** with `outboundTrafficPolicy: REGISTRY_ONLY` - Blocks unknown destinations
2. **ServiceEntry** - Registers allowed external services in the mesh registry

### 4.2 Three Levels of Egress Control

| Level | Sidecar Location | workloadSelector | Scope |
|-------|------------------|------------------|-------|
| **Pod-level** | App namespace | ✅ Has labels | Only pods matching labels |
| **Namespace-level** | App namespace | ❌ None | All pods in namespace |
| **Cluster-wide** | `istio-system` | ❌ None | All pods in entire mesh |

### 4.3 Cluster-Wide Egress Policy (Recommended)

```yaml
# egress/cluster-egress-policy.yaml
# Apply: kubectl apply -f egress/cluster-egress-policy.yaml
---
# 1. Register allowed external services
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: zenquotes-external
  namespace: istio-system          # istio-system for cluster-wide
  labels:
    managed-by: network-policies
spec:
  hosts:
    - zenquotes.io
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
  exportTo:
    - "*"                          # Visible to all namespaces

---
# 2. Register Kubernetes API (for kubectl/API access from pods)
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: kubernetes-api-external
  namespace: istio-system
  labels:
    managed-by: network-policies
spec:
  hosts:
    - api.c-XXXXX.kyma.ondemand.com    # Your Kyma API server
  ports:
    - number: 443
      name: https
      protocol: TLS
  location: MESH_EXTERNAL
  resolution: DNS
  exportTo:
    - "*"

---
# 3. Cluster-wide Sidecar - Block all unknown destinations
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: cluster-egress-policy
  namespace: istio-system          # istio-system = mesh-wide
  labels:
    managed-by: network-policies
spec:
  # No workloadSelector = applies to ALL pods in mesh
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY            # Block unknown destinations
  egress:
    - hosts:
        - "*/*"                    # All internal services (includes ServiceEntries)
```

### 4.4 Namespace-Level Egress Policy

```yaml
# egress/namespace-egress-policy.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: zenquotes-external
  namespace: garden                # Same namespace as pods
  labels:
    managed-by: network-policies
spec:
  hosts:
    - zenquotes.io
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: restrict-egress-namespace
  namespace: garden                # Applied to garden namespace
  labels:
    managed-by: network-policies
spec:
  # No workloadSelector = applies to ALL pods in garden namespace
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "garden/*"               # Internal services in garden
        - "istio-system/*"         # Istio control plane
        - "kube-system/*"          # DNS resolution
```

### 4.5 Pod-Level Egress Policy

```yaml
# egress/pod-egress-policy.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: zenquotes-external
  namespace: garden
  labels:
    managed-by: network-policies
spec:
  hosts:
    - zenquotes.io
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: restrict-egress-pod
  namespace: garden
  labels:
    managed-by: network-policies
spec:
  workloadSelector:
    labels:
      app.kubernetes.io/name: beautiful-garden    # ONLY this pod
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "garden/*"
        - "istio-system/*"
        - "kube-system/*"
```

### 4.6 What Changes from Default (Egress)

| Aspect | Default | After Policy | Why Change |
|--------|---------|--------------|------------|
| `outboundTrafficPolicy` | `ALLOW_ANY` | `REGISTRY_ONLY` | Block unknown external destinations |
| External URLs | All accessible | Only ServiceEntry hosts | Prevent data exfiltration |
| Unknown destinations | Permitted | Return 502/503 | Security - deny unknown targets |

---

## 4.7 ServiceEntry with IP Addresses

When the external service doesn't have a DNS name (e.g., a database or legacy system), you can use IP addresses directly in ServiceEntry.

### Synthetic Host (Mandatory but Non-Functional)

> [!IMPORTANT]
> **The `hosts` field is MANDATORY** in ServiceEntry, even when using IP addresses. However, this hostname is **NOT functional** for routing - it's only used as an identifier within the Istio mesh registry. The actual traffic is routed based on the `addresses` field.

| Field | Purpose | Required | Functional for Routing? |
|-------|---------|----------|------------------------|
| `hosts` | Mesh registry identifier | ✅ Mandatory | ❌ No (synthetic/arbitrary) |
| `addresses` | Actual IP(s) for traffic | ✅ For IP-based routing | ✅ Yes |
| `endpoints` | Target IPs for load balancing | ✅ For `STATIC` resolution | ✅ Yes |

### Example 1: Single IP Address (Database Server)

```yaml
# egress/external-database.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-database
  namespace: istio-system
  labels:
    managed-by: network-policies
spec:
  # ⚠️ MANDATORY: Arbitrary hostname - used ONLY as mesh registry identifier
  # This hostname is NOT used for actual DNS resolution or routing
  # You can use any naming convention, e.g., "service-name.internal"
  hosts:
    - external-db.internal         # Synthetic domain (pick any name you want)
  
  # ✅ ACTUAL ROUTING: Traffic matching this IP will be allowed
  addresses:
    - 203.0.113.50/32              # The real IP address of the database
  
  ports:
    - number: 5432
      name: postgres
      protocol: TCP
  
  location: MESH_EXTERNAL          # Outside the mesh
  
  # ✅ Use STATIC when providing explicit IP addresses
  # STATIC = use endpoints for routing (DNS is not queried)
  resolution: STATIC
  
  # ✅ Required for STATIC resolution - the actual target IPs
  endpoints:
    - address: 203.0.113.50        # Where traffic is actually sent

  exportTo:
    - "*"                          # Cluster-wide visibility
```

### Example 2: IP Range (CIDR Block)

```yaml
# egress/corporate-network.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: corporate-network
  namespace: istio-system
  labels:
    managed-by: network-policies
spec:
  # ⚠️ MANDATORY: Synthetic hostname (not functional)
  hosts:
    - "*.corp.internal"            # Arbitrary - won't be used for routing
  
  # ✅ ACTUAL ROUTING: Allow access to entire subnet
  addresses:
    - 10.100.0.0/16                # Corporate network CIDR
  
  ports:
    - number: 443
      name: https
      protocol: HTTPS
    - number: 80
      name: http
      protocol: HTTP
  
  location: MESH_EXTERNAL
  
  # ✅ Use NONE for CIDR - let the network handle routing
  # NONE = passthrough, don't intercept or resolve
  resolution: NONE                 # No endpoints needed for CIDR
  
  exportTo:
    - "*"
```

### Example 3: Multiple IP Addresses (Load Balanced)

```yaml
# egress/legacy-api-servers.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: legacy-api-servers
  namespace: istio-system
  labels:
    managed-by: network-policies
spec:
  # ⚠️ MANDATORY: Synthetic hostname for mesh registration
  hosts:
    - legacy-api.internal          # Identifier only - not used for routing
  
  # ✅ ACTUAL ROUTING: All these IPs are allowed
  addresses:
    - 192.168.1.10/32
    - 192.168.1.11/32
    - 192.168.1.12/32
  
  ports:
    - number: 8080
      name: http
      protocol: HTTP
  
  location: MESH_EXTERNAL
  resolution: STATIC
  
  # ✅ Endpoints for load balancing across multiple IPs
  endpoints:
    - address: 192.168.1.10
      weight: 33                   # Optional: load balancing weight
    - address: 192.168.1.11
      weight: 33
    - address: 192.168.1.12
      weight: 34
  
  exportTo:
    - "*"
```

### Resolution Modes Comparison

| `resolution` | When to Use | `addresses` | `endpoints` | DNS Lookup |
|--------------|-------------|-------------|-------------|------------|
| `DNS` | Hostname-based (normal) | Optional | Not needed | ✅ Yes |
| `STATIC` | IP-based routing | Required | Required | ❌ No |
| `NONE` | CIDR/passthrough | Required | Not needed | ❌ No |

> [!TIP]
> **Naming Convention**: Use a consistent pattern for synthetic hostnames, such as:
> - `<service-name>.internal` (e.g., `legacy-db.internal`)
> - `<service-name>.external` (e.g., `payments-api.external`)
> - `<ip-address>.ip.local` (e.g., `192-168-1-10.ip.local`)

---

## 4.8 Kubernetes NetworkPolicy vs. Istio Policies

### What is Kubernetes NetworkPolicy?

Kubernetes NetworkPolicy is a **native K8s resource** for controlling pod-to-pod traffic at the network layer (L3/L4). It requires a CNI plugin (like Calico or Cilium) that supports NetworkPolicy.

### Comparison: Istio vs. Kubernetes NetworkPolicy

| Aspect | Kubernetes NetworkPolicy | Istio Policies (AuthorizationPolicy, Sidecar) |
|--------|--------------------------|----------------------------------------------|
| **Layer** | L3/L4 (IP, port, protocol) | L7 (HTTP headers, paths, methods, JWT) |
| **Scope** | Pod-to-pod, namespace | Service mesh traffic (including external) |
| **Encryption** | None | mTLS automatic |
| **Requires** | CNI plugin (Calico, Cilium) | Istio sidecar injection |
| **External traffic** | Limited (ingress only) | Full control (ingress + egress) |
| **Observability** | None | Full tracing, metrics |

### Example: Kubernetes NetworkPolicy

```yaml
# k8s-network-policy/deny-all-egress.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: garden
spec:
  podSelector: {}                  # All pods in namespace
  policyTypes:
    - Egress
  egress: []                       # No egress allowed (empty = deny all)

---
# Allow only specific egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-internal
  namespace: garden
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    # Allow internal cluster traffic
    - to:
        - podSelector: {}          # Same namespace
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: istio-system
```

### Verdict: Do You Need Kubernetes NetworkPolicy in Kyma?

> [!NOTE]
> **Short Answer: Usually NO** - Istio policies are sufficient and more powerful for most use cases in Kyma.

| Scenario | NetworkPolicy Needed? | Why |
|----------|----------------------|-----|
| Standard Kyma workloads | ❌ No | Istio AuthorizationPolicy + Sidecar covers L7 |
| Pods without Istio sidecar | ✅ Yes | No Envoy = no Istio policy enforcement |
| Defense-in-depth compliance | ✅ Optional | Some audits require both layers |
| Non-HTTP protocols (raw TCP) | ⚠️ Maybe | Istio handles TCP, but NetworkPolicy adds L3 layer |
| Multi-tenant strict isolation | ✅ Recommended | Extra layer for namespace isolation |

### When to Use Both (Defense in Depth)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        DEFENSE IN DEPTH: Both Layers                             │
└─────────────────────────────────────────────────────────────────────────────────┘

  External Traffic
       │
       ▼
  ┌─────────────────────────────┐
  │  Kubernetes NetworkPolicy   │  ◄── L3/L4: IP/Port filtering
  │  (Calico/Cilium CNI)        │      Blocks traffic before it reaches pod
  └─────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────┐
  │  Istio AuthorizationPolicy  │  ◄── L7: HTTP method, path, headers
  │  (Envoy Sidecar)            │      Fine-grained access control
  └─────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────┐
  │        Your Pod              │
  └─────────────────────────────┘
```

### Final Recommendation for Kyma

| Environment | Recommendation |
|-------------|----------------|
| **Development** | Istio only (simpler) |
| **Production (standard)** | Istio only (sufficient) |
| **Production (regulated/compliance)** | Both Istio + NetworkPolicy |
| **Multi-tenant cluster** | Both (namespace isolation) |

> [!CAUTION]
> **Kyma CNI Limitation**: SAP BTP Kyma uses a managed Kubernetes cluster. The CNI may or may not support NetworkPolicy depending on the underlying infrastructure. Check with `kubectl get nodes -o wide` and verify CNI plugin before relying on NetworkPolicy.

```bash
# Check if NetworkPolicy is supported
kubectl get networkpolicies -A

# If no errors and you can create policies, CNI supports it
# If you get errors or policies don't take effect, CNI doesn't support it
```

---

## 5. ExternalTrafficPolicy Deep Dive

### 5.1 What is ExternalTrafficPolicy?

A Kubernetes Service configuration that controls how traffic from external sources is routed to pods, specifically whether SNAT (Source Network Address Translation) is applied.

### 5.2 Two Modes Compared

| Aspect | `Cluster` (Default) | `Local` |
|--------|---------------------|---------|
| **Traffic routing** | Can route to any node | Only routes to local node |
| **Source IP** | Lost (replaced by node IP) | Preserved (client's real IP) |
| **Load balancing** | Even distribution | Uneven (depends on pod placement) |
| **Required for IP whitelist?** | ❌ No | ✅ Yes |

### 5.3 How Traffic Flows with Each Mode

#### Mode: `Cluster` (Default) - Source IP Lost

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    externalTrafficPolicy: Cluster (DEFAULT)                      │
└─────────────────────────────────────────────────────────────────────────────────┘

Client IP: 163.223.48.163

   Client                 Node A                    Node B                   Pod
     │                      │                         │                       │
     │  Request             │                         │                       │
     ├─────────────────────►│                         │                       │
     │                      │                         │                       │
     │                      │  kube-proxy: Forward    │                       │
     │                      │  (Pod is on Node B)     │                       │
     │                      ├────────────────────────►│                       │
     │                      │                         │                       │
     │                      │                         │  SNAT: Replace source │
     │                      │                         │  IP with Node A IP    │
     │                      │                         ├──────────────────────►│
     │                      │                         │                       │
     │                      │                         │  Pod sees: 10.0.1.5   │
     │                      │                         │  (Node A's IP)        │
     │                      │                         │  NOT: 163.223.48.163  │
```

> [!WARNING]
> With `Cluster` mode, `remoteIpBlocks` in AuthorizationPolicy will match the **Node IP** (e.g., `10.0.1.5`), not the client IP. Your whitelist won't work!

#### Mode: `Local` - Source IP Preserved

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      externalTrafficPolicy: Local                                │
└─────────────────────────────────────────────────────────────────────────────────┘

Client IP: 163.223.48.163

   Client                 Node A                    Node B                   Pod
     │                      │                         │  (Pod is here)         │
     │                      │                         │                       │
     │  Request             │                         │                       │
     ├─────────────────────►│                         │                       │
     │                      │                         │                       │
     │                      │  kube-proxy: Node A     │                       │
     │                      │  has no local pods,     │                       │
     │                      │  drop the packet OR     │                       │
     │                      │  LB health check fails  │                       │
     │                      │  (NLB won't send here)  │                       │
     │                      │                         │                       │
     │  Request             │                         │                       │
     ├───────────────────────────────────────────────►│                       │
     │                      │                         │                       │
     │                      │                         │  No SNAT!             │
     │                      │                         │  Preserve source IP   │
     │                      │                         ├──────────────────────►│
     │                      │                         │                       │
     │                      │                         │  Pod sees: 163.223.x  │
     │                      │                         │  (Real client IP)     │
```

### 5.4 How to Configure ExternalTrafficPolicy

```bash
# Check current setting
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.externalTrafficPolicy}'

# Patch to Local (REQUIRED for IP whitelisting)
kubectl patch svc istio-ingressgateway -n istio-system \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

Or as YAML:

```yaml
# istio-ingress-service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
spec:
  externalTrafficPolicy: Local     # Preserve client IP
```

> [!IMPORTANT]
> **Kyma-specific**: In some Kyma versions, the ingress gateway service may be managed. Check if patching is allowed or if you need to use a configuration profile.

---

## 6. NATS Messaging in Kyma

### 6.1 What is NATS?

NATS is a messaging system used by Kyma for **event-driven architecture**. When you create Subscriptions to listen for events (e.g., from SAP Commerce Cloud), NATS handles the message routing.

### 6.2 NATS Architecture in Kyma

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              NATS in Kyma                                        │
└─────────────────────────────────────────────────────────────────────────────────┘

   External Event Source          NATS Cluster              Your Subscriber Pod
   (SAP Commerce Cloud)           (kyma-system)             (garden namespace)
         │                            │                            │
         │ Event: order.created       │                            │
         ├───────────────────────────►│                            │
         │                            │                            │
         │                            │  Route to subscriber       │
         │                            ├───────────────────────────►│
         │                            │                            │
         │                            │                            │ Process event
         │                            │                            │
```

### 6.3 Network Policy Considerations for NATS

When restricting egress, you must allow traffic to NATS:

```yaml
# egress/nats-egress.yaml
---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: allow-nats-egress
  namespace: garden
  labels:
    managed-by: network-policies
spec:
  workloadSelector:
    labels:
      app.kubernetes.io/name: beautiful-garden
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "garden/*"               # Internal services
        - "istio-system/*"         # Istio control plane
        - "kube-system/*"          # DNS
        - "kyma-system/*"          # NATS and Kyma components
```

> [!NOTE]
> NATS traffic is internal, so it's covered by `kyma-system/*` in the egress hosts. You don't need a ServiceEntry for NATS.

### 6.4 Default NATS Configuration

| Setting | Default Value | Impact |
|---------|---------------|--------|
| NATS port | 4222 | Internal communication |
| JetStream | Enabled | Persistent messaging |
| Namespace | `kyma-system` | Must allow egress to this namespace |

---

## 7. AWS Load Balancer Considerations

### 7.1 How Kyma Uses AWS NLB

SAP BTP Kyma on AWS uses a **Network Load Balancer (NLB)** to expose the Istio Ingress Gateway externally.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        AWS NLB → Kyma Cluster                                    │
└─────────────────────────────────────────────────────────────────────────────────┘

   Internet           AWS NLB              Target Group           Istio Gateway
                      (Elastic IP)         (Node Ports)           (Pod)
       │                  │                    │                      │
       │  HTTPS           │                    │                      │
       ├─────────────────►│                    │                      │
       │                  │                    │                      │
       │                  │  Health checks     │                      │
       │                  ├───────────────────►│                      │
       │                  │                    │                      │
       │                  │  Forward to        │                      │
       │                  │  healthy targets   │                      │
       │                  ├───────────────────►│                      │
       │                  │                    │                      │
       │                  │                    │  Route to pod        │
       │                  │                    ├─────────────────────►│
       │                  │                    │                      │
```

### 7.2 NLB IP Preservation

AWS NLB in **TCP mode** (which Kyma uses) preserves the client source IP by default. However, this only works end-to-end if:

1. NLB preserves client IP (default ✅)
2. Target group uses instance mode (not IP mode)
3. `externalTrafficPolicy: Local` is set on the Service

### 7.3 Finding the NLB IP

```bash
# Get the NLB hostname
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Resolve to IPs
nslookup <hostname>

# Example output:
# Name:    a1b2c3d4e5f6g7h8.elb.eu-central-1.amazonaws.com
# Address: 3.120.xxx.xxx
# Address: 18.195.xxx.xxx
```

### 7.4 NLB with Target Group Health Checks

When using `externalTrafficPolicy: Local`:

```yaml
# AWS annotations for optimal health checks
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  annotations:
    # Health check path
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: /healthz/ready
    # Health check port
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "15021"
    # Healthy threshold
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
spec:
  externalTrafficPolicy: Local
```

---

## 8. Complete Example Scenarios

### Scenario 1: Production Cluster with Full Security

**Requirements:**
- Only allow office IPs (163.223.48.163, 103.99.8.235) to access the cluster
- Only allow pods to reach zenquotes.io and internal services
- Preserve client IPs for audit logging

**Step 1: Configure ExternalTrafficPolicy**
```bash
kubectl patch svc istio-ingressgateway -n istio-system \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

**Step 2: Apply Ingress Policy**
```yaml
# production-ingress.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: production-ip-whitelist
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
    - from:
        - source:
            remoteIpBlocks:
              - "163.223.48.163/32"    # Office 1
              - "103.99.8.235/32"      # Office 2
```

**Step 3: Apply Egress Policy**
```yaml
# production-egress.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: zenquotes-external
  namespace: istio-system
spec:
  hosts:
    - zenquotes.io
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
  exportTo:
    - "*"

---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: production-egress-policy
  namespace: istio-system
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "*/*"
```

**Apply:**
```bash
kubectl apply -f production-ingress.yaml
kubectl apply -f production-egress.yaml
```

---

### Scenario 2: Development Cluster with Relaxed Egress

**Requirements:**
- Restrict ingress to team IPs
- Allow all egress (developers need to test various APIs)

**Configuration:**
```yaml
# dev-policies.yaml
---
# Ingress: Restricted
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: dev-ip-whitelist
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
    - from:
        - source:
            remoteIpBlocks:
              - "0.0.0.0/0"     # All IPs (for dev only!)

---
# Egress: Not restricted (no Sidecar with REGISTRY_ONLY)
# By default, Istio uses ALLOW_ANY for outbound traffic
```

> [!WARNING]
> Never use `0.0.0.0/0` in production. This example is for development clusters ONLY.

---

### Scenario 3: Multi-Tenant Cluster with Namespace Isolation

**Requirements:**
- `team-a` namespace can only reach their own services and `payments-api.com`
- `team-b` namespace can only reach their own services and `analytics-api.com`

```yaml
# team-a-policies.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: payments-api
  namespace: team-a
spec:
  hosts:
    - payments-api.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: team-a-egress
  namespace: team-a
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "team-a/*"           # Only team-a services
        - "istio-system/*"     # Control plane
        - "kube-system/*"      # DNS
```

```yaml
# team-b-policies.yaml
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: analytics-api
  namespace: team-b
spec:
  hosts:
    - analytics-api.com
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: team-b-egress
  namespace: team-b
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
  egress:
    - hosts:
        - "team-b/*"           # Only team-b services
        - "istio-system/*"     # Control plane
        - "kube-system/*"      # DNS
```

---

## 9. Quick Reference Commands

### Check Current Configuration

```bash
# Ingress policies
kubectl get authorizationpolicy -n istio-system
kubectl get authorizationpolicy -A

# Egress policies
kubectl get sidecar -A
kubectl get serviceentry -A

# ExternalTrafficPolicy
kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.externalTrafficPolicy}'
```

### Apply Policies

```bash
# Apply all ingress policies
kubectl apply -f network-policies/apply/ingress/

# Apply all egress policies
kubectl apply -f network-policies/apply/egress/

# Set externalTrafficPolicy to Local
kubectl patch svc istio-ingressgateway -n istio-system \
  -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

### Test Policies

```bash
# Test ingress (from allowed IP)
curl -I https://your-app.kyma.ondemand.com

# Test egress from inside pod
kubectl exec -n garden deploy/beautiful-garden -c beautiful-garden -- \
  wget -qO- --timeout=5 https://zenquotes.io/api/random

# Test blocked egress
kubectl exec -n garden deploy/beautiful-garden -c beautiful-garden -- \
  wget -qO- --timeout=5 https://google.com 2>&1 || echo "BLOCKED ✅"
```

### Troubleshooting

```bash
# Check Envoy config for a pod
kubectl exec -n garden deploy/beautiful-garden -c istio-proxy -- \
  pilot-agent request GET config_dump > envoy-config.json

# Check if traffic policy is being applied
istioctl analyze -n garden

# View Sidecar effective configuration
istioctl proxy-config listener deploy/beautiful-garden -n garden
```

### Delete Policies

```bash
# Remove all ingress restrictions
kubectl delete authorizationpolicy -n istio-system --all

# Remove all egress restrictions
kubectl delete sidecar -n istio-system -l managed-by=network-policies
kubectl delete serviceentry -n istio-system -l managed-by=network-policies
```

---

## Summary: Default vs. Secured

| Configuration | Default | Secured | Command to Apply |
|---------------|---------|---------|------------------|
| Ingress IPs | All allowed | Whitelist only | `kubectl apply -f ingress/` |
| Egress destinations | All allowed | ServiceEntry only | `kubectl apply -f egress/` |
| ExternalTrafficPolicy | `Cluster` | `Local` | `kubectl patch svc ...` |
| Source IP visible | No (SNAT) | Yes (preserved) | Via `Local` policy |

> [!TIP]
> Always test policies in a non-production environment first. A misconfigured policy can lock you out of the cluster!
