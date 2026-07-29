# k8s-studylab

A single-cluster Kubernetes **studylab** — a deliberate learning environment for the
stack I use professionally: k3s, Istio, GitOps, progressive delivery, observability,
and event-driven autoscaling.

Not a homelab. Nothing here is meant to run anything useful — the goal is to build
each piece by hand, understand *why* it exists, and prove it works.

Everything runs on one MacBook via **Rancher Desktop**. `staging` and `prod` are
namespaces in a single cluster, not separate clusters.

---

## Status

| Phase | Focus | State |
|-------|-------|-------|
| 0 | Cluster base, core objects, probes | ✅ done |
| 1 | Namespaces, RBAC, **NetworkPolicy isolation** | ✅ done |
| 2 | Istio ingress + real TLS on `*.rcamine.com` | ✅ done |
| 3 | CI: build → GHCR | 🚧 in progress |
| 4 | GitOps with Argo CD | ⬜ |
| 5 | Canary 1→100%, Prometheus-gated (Argo Rollouts) | ⬜ |
| 6 | Prometheus + Grafana + Loki, dashboards | ⬜ |
| 7 | HPA → KEDA on Kafka consumer lag | ⬜ |
| 8 | Sealed Secrets, Velero, mTLS STRICT, tracing | ⬜ |

---

## Architecture

```mermaid
flowchart LR
    U[Browser] -->|HTTPS 443| GW[Istio ingress gateway<br/>istio-system]
    GW -->|VirtualService| WS[web · staging]
    GW -.->|planned: apex| WP[web · prod]

    CM[cert-manager] -->|DNS-01| CF[(Cloudflare DNS)]
    CM -->|wildcard cert| GW

    GH[GitHub Actions] -->|push image| GHCR[(ghcr.io)]
    GHCR -.->|pull| WS
    GHCR -.->|pull| WP

    WS <-.->|BLOCKED by NetworkPolicy| WP
```

**Traffic path:** browser → Istio ingress gateway (terminates TLS using a Let's Encrypt
wildcard cert) → VirtualService routes by `Host` header → app pod in `staging`.

**Isolation:** both `staging` and `prod` run default-deny ingress plus an additive
allow-rule for same-namespace, `monitoring`, and `istio-system` traffic. Cross-env
traffic is dropped in both directions — verified, not assumed.

---

## Layout

```
00-namespaces.yaml        namespaces + istio-injection labels
10-test-apps.yaml         nginx placeholder workloads (being replaced by app/)
20-networkpolicy.yaml     default-deny ingress (prod)
21-allow-ingress.yaml     additive allow-rules (prod)
22-staging-policies.yaml  same pair for staging
30-istio-ingress.yaml     Gateway + VirtualService for staging.rcamine.com
40-clusterissuers.yaml    Let's Encrypt staging + prod ACME issuers
41-certificate.yaml       wildcard *.rcamine.com + apex
cert-manager-values.yaml  Helm values (DNS-01 recursive nameservers)
app/                      the demo Go service
.github/workflows/        CI: build + push to GHCR
BOOTSTRAP.md              manual steps to recreate this from zero
```

Manifests are numbered by apply order — lower numbers are prerequisites for higher ones.

---

## The demo app

A ~40-line Go HTTP server (`app/`) that reports its version and pod name:

```
$ curl https://staging.rcamine.com/
hello from web a1b2c3d (pod web-6dccb9bf47-t8j8j)
```

Deliberately minimal, but with the three properties the later phases need:

- **version in the response** — makes canary traffic-splitting visible
- **`/healthz`** — backs liveness/readiness probes; readiness is the first gate a canary must pass
- **graceful shutdown on SIGTERM** — drains in-flight requests instead of dropping them
  when pods are killed during a rollout

Built multi-stage into a distroless image: **~3 MB**, no shell, non-root.

---

## Stack

| Component | Version | Install |
|---|---|---|
| k3s (Rancher Desktop) | `v1.36.2+k3s1` | Rancher Desktop, Traefik disabled |
| Istio | `1.30.2` | sidecar mode, PERMISSIVE mTLS |
| cert-manager | `v1.21.0` | Helm |
| Go | `1.26` | — |

External dependencies are only **GitHub** (repo + Actions + GHCR) and **rcamine.com**
(registered at Namecheap, DNS delegated to Cloudflare). Everything else is self-hosted
and free. Terraform is intentionally out of scope — provisioning is done by hand so the
focus stays on what runs *inside* the cluster.

---

## Verifying it works

```bash
# HTTPS with a real, browser-trusted cert
curl -sI --resolve staging.rcamine.com:443:127.0.0.1 https://staging.rcamine.com/

# HTTP redirects to HTTPS
curl -so /dev/null -w '%{http_code}\n' -H 'Host: staging.rcamine.com' http://localhost/   # 301

# isolation holds: staging cannot reach prod
kubectl exec -n staging deploy/client -- curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://web.prod   # 000
```

> **Testing NetworkPolicy:** use a long-lived pod and `kubectl exec`, never a throwaway
> `kubectl run --rm`. k3s's policy controller takes several seconds to program a new
> pod's IP into the allow-ipset, so a fast probe races ahead of it and reports a false
> block.

## Known lab quirks

**Istio 503s after the laptop sleeps.** Sidecar mTLS certs are ~24h and only rotate
while connected to istiod. After a multi-day suspend they expire and the gateway logs
`CERTIFICATE_VERIFY_FAILED ... certificate has expired`. mTLS is mutual, so restart
both ends:

```bash
kubectl rollout restart deploy/web deploy/client -n staging
kubectl rollout restart deploy/web -n prod
kubectl rollout restart deploy/istio-ingressgateway deploy/istio-egressgateway -n istio-system
```

This is an artifact of running on a laptop — a cluster that stays online rotates these
silently. Unrelated to the public Let's Encrypt cert, which is valid for 90 days and
auto-renews around day 60.
