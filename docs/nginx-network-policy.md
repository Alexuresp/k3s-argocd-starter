# Nginx Network Policy Help

## Overview

`my-apps/nginx/network-policy.yaml` defines a **Layer 3/4 CiliumNetworkPolicy** for the `nginx-example` namespace.

## What it allows

| Direction | Source / Destination | Ports | Purpose |
|-----------|----------------------|-------|---------|
| Ingress | `fromEntities: ingress` | TCP/80 | Traffic from Cilium Gateway only |
| Egress | `toEntities: world` | all | Internet access (HTTP/HTTPS, apt, wget, curl) |
| Egress | `toEndpoints: kube-dns` | UDP/TCP 53 | DNS resolution (internal + external) |

## Why `fromEntities: ingress` instead of namespace selectors

- The Cilium Gateway proxy (`cilium-envoy-*`) runs in **host-network / node-level mode**.
- It often does **not** appear as a normal Cilium endpoint with a standard namespace identity.
- Selectors like `io.kubernetes.pod.namespace: kube-system` or `k8s-app: cilium-envoy` may **not match**.
- `ingress` entity covers Gateway proxy traffic without opening the app to all cluster pods.

## What is blocked

- Direct pod-to-pod access to nginx from other namespaces (e.g., `default`, `searxng`, `argocd`)
- Direct access to the ClusterIP (`10.43.x.x`) from outside the Gateway path

## Verification

```bash
# Should work: external traffic via Gateway
curl http://nginx.192.168.1.18.traefik.me/

# Should be blocked: direct access from another namespace
kubectl exec -n default curl-test -- curl -s --connect-timeout 3 http://10.43.241.29/
```

## Extending to Layer 7

This policy covers only L3/L4. For HTTP-level controls (paths, methods, headers), you have two options:

1. **L7 CiliumNetworkPolicy** with `toPorts.http` rules inside the same policy.
2. **CiliumEnvoyConfig** — attach an Envoy sidecar to the nginx pod and apply L7 rules via xDS.

Example L7 extension:
```yaml
ingress:
  - fromEntities:
      - ingress
    toPorts:
      - ports:
          - port: "80"
            protocol: TCP
        rules:
          http:
            - method: GET
              path: "/static/*"
```

## Troubleshooting

- If nginx returns 503 after applying the policy, check:
  ```bash
  kubectl exec -n kube-system cilium-<pod> -- cilium-dbg policy get
  ```
- If DNS fails inside nginx, verify CoreDNS labels:
  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[*].metadata.name}'
  ```
