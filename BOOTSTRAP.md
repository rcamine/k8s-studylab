# Bootstrap

The manual steps required to recreate this lab from zero.

Everything else in this repo is declarative — `kubectl apply` and it converges. The
steps below are the ones that **can't** be, because they either create the thing that
runs the automation, involve a credential that must be typed by a human, or hit a
provider that has no API for the operation.

Keeping this list short and honest is the point. Anything here that *could* be
declarative should eventually move out of this file.

---

## 1. Rancher Desktop + k3s

Install Rancher Desktop and enable Kubernetes.

**Disable the bundled Traefik** so Istio can own ingress. In Rancher Desktop:
*Preferences → Kubernetes → uncheck "Traefik"*, or pass `--disable=traefik` to k3s.

Give the VM as much RAM as the Mac can spare (leave macOS ~4-6 GB). Istio sidecars,
the monitoring stack, Argo, and Kafka add up fast on 16 GB.

> **Why manual:** this creates the cluster. Nothing in-cluster can bootstrap itself.

---

## 2. Domain + DNS delegation

`rcamine.com` is registered at **Namecheap**, with nameservers pointed at **Cloudflare**.
Cloudflare hosts the zone; cert-manager talks to Cloudflare's API to solve ACME DNS-01
challenges.

> **Why manual:** domain registration is a purchase, and changing nameservers at the
> registrar is a one-time account action. Terraform *could* manage the Cloudflare zone
> once delegated — deliberately out of scope here.

---

## 3. Istio

Installed in sidecar mode (matching the setup this lab is modelling).

```bash
istioctl install --set profile=demo -y
```

Namespaces opt into injection with the `istio-injection=enabled` label — see
`00-namespaces.yaml`. Keep it off everywhere except `staging` and `prod`; every meshed
pod costs ~50-100 Mi.

---

## 4. cert-manager

```bash
kubectl apply -f 00-namespaces.yaml     # creates the cert-manager namespace
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager \
  -n cert-manager -f cert-manager-values.yaml
```

`cert-manager-values.yaml` sets `--dns01-recursive-nameservers-only` with public
resolvers. **This is required, not cosmetic:** by default cert-manager's DNS-01
self-check uses cluster CoreDNS (`10.43.0.10`), which can't resolve the public SOA
record for `rcamine.com`. Without it, challenges hang on
`Could not find the SOA record`.

---

## 5. Cloudflare API token → Secret

Create a token at Cloudflare → *My Profile → API Tokens*, scoped as narrowly as
possible: **Zone → DNS → Edit**, restricted to the `rcamine.com` zone only.

```bash
read -rs CF_TOKEN && echo "captured ${#CF_TOKEN} chars"   # verify it's non-zero!
kubectl create secret generic cloudflare-api-token \
  -n cert-manager --from-literal=api-token="$CF_TOKEN"
unset CF_TOKEN
```

**This value must never be written to a file in this repo.** It's created directly in
the cluster, and the manifests only reference it by name. The `echo "${#CF_TOKEN}"`
check exists because a silent empty paste produces a 0-byte Secret and a
`no Cloudflare credential has been given` error that looks nothing like the real cause.

> **Why manual:** a long-lived credential typed by a human. Phase 8 (Sealed Secrets /
> External Secrets) is precisely the technique for getting this into git safely.

---

## 6. Apply the manifests

In numeric order — the numbering is the dependency order:

```bash
kubectl apply -f 00-namespaces.yaml
kubectl apply -f 10-test-apps.yaml
kubectl apply -f 20-networkpolicy.yaml -f 21-allow-ingress.yaml -f 22-staging-policies.yaml
kubectl apply -f 40-clusterissuers.yaml
kubectl apply -f 41-certificate.yaml     # wait for READY=True before the Gateway
kubectl apply -f 30-istio-ingress.yaml
```

Wait for the certificate before applying the Gateway — it references the Secret
cert-manager writes:

```bash
kubectl wait --for=condition=Ready certificate/wildcard-rcamine -n istio-system --timeout=5m
```

> Note: the cert Secret must live in `istio-system` (the gateway's namespace), not the
> app's namespace. `credentialName` is resolved relative to the gateway.

---

## 7. GHCR package visibility — nothing to do (if the repo is public)

A package published from a **public** repo via the workflow's `GITHUB_TOKEN` inherits
that visibility, so it's pullable anonymously and needs no `imagePullSecrets`. Verified:

```bash
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:rcamine/studylab-web:pull&service=ghcr.io" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["token"])')
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  https://ghcr.io/v2/rcamine/studylab-web/manifests/latest        # 200 = public
```

**If the package ever *is* private**, it must be changed in the UI —
*Repo → Packages → Package settings → Change visibility*. GitHub exposes no REST
endpoint for this: the packages API can list, get, delete, and restore, but not set
visibility. And `GITHUB_TOKEN` couldn't do it anyway — it's scoped to repo contents and
packages, not account administration, so automating it would mean storing a long-lived
admin PAT in CI. Worse trade than one click.

---

## 8. Local DNS for browser testing

The cluster isn't reachable from the public internet, so point the hostnames at the
local ingress:

```
# /etc/hosts
127.0.0.1  staging.rcamine.com
127.0.0.1  rcamine.com
```

`curl --resolve staging.rcamine.com:443:127.0.0.1 https://...` does the same thing
without touching `/etc/hosts`.

> **Why manual:** the real DNS records intentionally don't point at a laptop.

---

## Still to come

- **Argo CD** (Phase 4) must be hand-installed — it can't deploy itself. The
  **app-of-apps** pattern minimizes this to a single manual `Application` that points
  at a directory of all the others; everything after that is git-driven.
