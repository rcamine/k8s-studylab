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
| 3 | CI: build → GHCR → running in-cluster | ✅ done |
| 4 | GitOps with Argo CD | ✅ done |
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

    GIT[(git: main)] -->|polls| ACD[Argo CD<br/>argocd]
    ACD -->|auto-sync + selfHeal| WS
    ACD -->|manual sync| WP

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
root.yaml                 app-of-apps — the ONE hand-applied manifest
argocd/
  infra.yaml              Application → infra/
  staging.yaml            Application → staging/   (automated + selfHeal)
  prod.yaml               Application → prod/      (manual)
infra/
  namespaces.yaml         namespaces + istio-injection labels
  clusterissuers.yaml     Let's Encrypt staging + prod ACME issuers
  certificate.yaml        wildcard *.rcamine.com + apex (in istio-system)
staging/
  web.yaml                Deployment + Service
  client.yaml             netshoot pod to curl from
  networkpolicy.yaml      default-deny ingress + additive allow-rules
  istio-ingress.yaml      Gateway + VirtualService for staging.rcamine.com
prod/
  web.yaml                Deployment + Service
  networkpolicy.yaml      default-deny ingress + additive allow-rules
cert-manager-values.yaml  Helm values (DNS-01 recursive nameservers)
app/                      the demo Go service
.github/workflows/        CI: build + push to GHCR
BOOTSTRAP.md              manual steps to recreate this from zero
```

One directory per Argo Application. The split is what gives each environment its
own sync, history, and rollback — a broken `prod/` manifest can't block a staging
deploy, and `argocd app rollback staging` touches staging alone.

> **No Kustomize, deliberately.** `staging/web.yaml` and `prod/web.yaml` are
> near-identical copies. A Kustomize `base/` + overlays would remove that
> duplication, at the cost of a rendering step between what's in git and what
> lands in the cluster. For now the manifests are literal and greppable; revisit
> when maintaining two copies actually hurts.

---

## The demo app

A ~40-line Go HTTP server (`app/`) that reports its version and pod name:

```
$ curl https://staging.rcamine.com/
hello from web 6e6a4cc (pod web-64b8d66fcb-smj4x)
```

Deliberately minimal, but with the three properties the later phases need:

- **version in the response** — makes canary traffic-splitting visible
- **`/healthz`** — backs liveness/readiness probes; readiness is the first gate a canary must pass
- **graceful shutdown on SIGTERM** — drains in-flight requests instead of dropping them
  when pods are killed during a rollout

Built multi-stage into a distroless image: **~3 MB**, no shell, non-root.

> Distroless means there's no shell in the pod, so `kubectl exec` won't work. Use
> `kubectl debug -n staging <pod> --image=nicolaka/netshoot --target=web` to attach an
> ephemeral container instead of baking tools into the image.

---

## Delivery pipeline

```
git push  →  GitHub Actions  →  GHCR  →  k3s pulls  →  Istio  →  https://staging.rcamine.com
```

`.github/workflows/build.yml` builds `app/` on every push that touches it and pushes to
`ghcr.io/rcamine/studylab-web`, tagged with the **full commit SHA**. Deployments pin
that SHA — never `:latest`, which is a mutable pointer that moves under you and makes
"what is actually deployed?" unanswerable.

A few choices worth noting:

- **`runs-on: ubuntu-24.04-arm`** — a native arm64 runner (free on public repos), so the
  image matches Apple Silicon without slow QEMU emulation.
- **`permissions: packages: write`** — workflow tokens are read-only by default; without
  this the push fails with an unhelpful 403.
- **`secrets.GITHUB_TOKEN`** — minted per run and discarded. No PAT to create or rotate.
- **`paths: app/**`** — editing a manifest doesn't trigger an image build.

The package inherits the repo's public visibility, so the cluster pulls anonymously and
needs no `imagePullSecrets`.

---

## GitOps

Nothing here is deployed with `kubectl apply`. Argo CD polls this repo and reconciles
the cluster to match it — a deploy is a commit.

```
git push  →  root (auto-sync)  →  argocd/*.yaml  →  infra / staging / prod
```

`root.yaml` is the **only** manifest applied by hand, and only once. It syncs `argocd/`,
so the Applications themselves are git-managed; adding an environment is a new file plus
a push. It lives at the repo root rather than inside `argocd/` on purpose — an
Application that manages itself can be left unable to sync its own fix.

| Application | Path | Sync |
|---|---|---|
| `root` | `argocd/` | automated, `selfHeal`, **no prune** |
| `infra` | `infra/` | manual |
| `staging` | `staging/` | automated, `selfHeal`, `prune` |
| `prod` | `prod/` | manual |

**Why staging and prod differ.** `selfHeal` reverts any live change contradicting git
within seconds. In staging that's the point — drift becomes impossible and git is the
only way in. In prod it's hostile: during an incident you scale something by hand to
shed load, and an invisible controller silently undoes your intervention while you debug
why it "didn't work." Promotion to prod should be a decision, not a consequence of
pushing to `main`.

**Two deliberate safety choices:**

- **No prune on `root`.** Pruning there wouldn't delete a Deployment — it would delete an
  entire *Application*, and with cascade semantics everything beneath it.
- **Namespaces carry `argocd.argoproj.io/sync-options: Prune=false,Delete=false`.**
  Namespace deletion is recursive and ignores Argo's ownership model, so removing one
  takes every object inside — including things Argo never managed: the hand-created
  `cloudflare-api-token` Secret, the Let's Encrypt account keys, and (for `argocd`) Argo
  CD itself. Losing the ACME account key plus the cert can mean a **week locked out of
  HTTPS** on Let's Encrypt's duplicate-certificate rate limit, which no command fixes.

> `argocd app delete` **cascades by default** and deletes every managed resource.
> `kubectl delete app` does not, unless the `resources-finalizer.argocd.argoproj.io`
> finalizer is set. When restructuring — deleting an Application while keeping the
> workloads running — use `kubectl`, whose failure mode is harmless.

Useful commands:

```bash
argocd app diff staging       # what would change, against the normalized live object
argocd app history staging    # deploy log, each entry keyed to a git SHA
argocd app rollback staging 1 # revert to a previous synced revision
```

> Argo compares **content, not revision strings**. A manually-synced app can sit at an
> older `.status.sync.revision` and still be `Synced` — it just means nothing in its
> directory changed.

---

## Stack

| Component | Version | Install |
|---|---|---|
| k3s (Rancher Desktop) | `v1.36.2+k3s1` | Rancher Desktop, Traefik disabled |
| Istio | `1.30.2` | sidecar mode, PERMISSIVE mTLS |
| cert-manager | `v1.21.0` | Helm |
| Argo CD | `v3.4.5` | `kubectl apply --server-side` (see BOOTSTRAP) |
| Go | `1.26` | — |

External dependencies are only **GitHub** (repo + Actions + GHCR) and **rcamine.com**
(registered at Namecheap, DNS delegated to Cloudflare). Everything else is self-hosted
and free. Terraform is intentionally out of scope — provisioning is done by hand so the
focus stays on what runs *inside* the cluster.

---

## Verifying it works

```bash
# the whole pipeline in one line: our image, our cert, through Istio
curl -s --resolve staging.rcamine.com:443:127.0.0.1 https://staging.rcamine.com/
# -> hello from web <short-sha> (pod web-xxxxxxxxx-xxxxx)

# HTTPS with a real, browser-trusted cert
curl -sI --resolve staging.rcamine.com:443:127.0.0.1 https://staging.rcamine.com/

# HTTP redirects to HTTPS
curl -so /dev/null -w '%{http_code}\n' -H 'Host: staging.rcamine.com' http://localhost/   # 301

# isolation holds: staging cannot reach prod
kubectl exec -n staging deploy/client -- curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://web.prod

# GitOps: everything reconciled, nothing drifted
kubectl get app -n argocd     # root/infra/staging/prod all Synced + Healthy

# selfHeal works: this change is reverted within seconds
kubectl scale deploy/web -n staging --replicas=5
kubectl get deploy web -n staging -o jsonpath='{.spec.replicas}{"\n"}'   # back to 2
```

> That last one returns **503**, not 200 — the client's Envoy sidecar reporting
> `upstream connect error ... remote connection failure` because NetworkPolicy dropped
> the packets. (Without a sidecar in the path it shows as `000`, a curl timeout.) Either
> way it's blocked; a successful reply would have been 200.

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
