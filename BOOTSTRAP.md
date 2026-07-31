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
`infra/namespaces.yaml`. Keep it off everywhere except `staging` and `prod`; every meshed
pod costs ~50-100 Mi.

---

## 4. cert-manager

```bash
kubectl apply -f infra/namespaces.yaml     # creates the cert-manager namespace
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

In dependency order — infra first, then the environments:

```bash
kubectl apply -f infra/namespaces.yaml
kubectl apply -f infra/clusterissuers.yaml
kubectl apply -f infra/certificate.yaml   # wait for READY=True before the Gateway
kubectl apply -R -f staging/
kubectl apply -R -f prod/
```

Wait for the certificate before applying the Gateway — it references the Secret
cert-manager writes:

```bash
kubectl wait --for=condition=Ready certificate/wildcard-rcamine -n istio-system --timeout=5m
```

> Note: the cert Secret must live in `istio-system` (the gateway's namespace), not the
> app's namespace. `credentialName` is resolved relative to the gateway.

> **This step is only for a cold rebuild.** Once Argo CD is running (step 7) it owns
> these directories — do not `kubectl apply` them by hand afterwards, or you'll be
> fighting `selfHeal` on staging.

---

## 7. Argo CD

Argo CD can't deploy itself, so this install is manual. Everything after it is git-driven.

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**`--server-side` is mandatory, not a preference.** Plain `kubectl apply` records the
manifest it applied into a `kubectl.kubernetes.io/last-applied-configuration` annotation,
and annotations are capped at 262,144 bytes. The `applicationsets.argoproj.io` CRD is
**~1.39 MB** of OpenAPI schema, so it is silently rejected while everything else
succeeds. The symptom is baffling: 6 of 7 pods healthy, and
`argocd-applicationset-controller` crash-looping every ~2 minutes with

```
no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

Server-side apply tracks ownership in `managedFields` instead, so there's no oversized
annotation. `--force-conflicts` is needed if anything was previously applied
client-side. Verify all three CRDs landed:

```bash
kubectl get crd | grep argoproj    # applications, applicationsets, appprojects
```

> If you hit the crash loop after fixing the CRD, delete the pod rather than waiting —
> `CrashLoopBackOff` has an exponential timer, so a fixed root cause still looks broken
> for minutes.

### Turn off Argo's own TLS

`infra/argocd-ingress.yaml` puts Argo behind the Istio gateway, which terminates TLS
with the Let's Encrypt wildcard cert. But `argocd-server` also serves TLS by default —
with a self-signed cert — so without this patch you get **double TLS**: the gateway would
have to re-encrypt to a certificate it has no reason to trust.

```bash
kubectl patch cm argocd-cmd-params-cm -n argocd --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deploy/argocd-server -n argocd
```

**This is not in git**, and it's the easiest step to forget on a rebuild. `argocd-cmd-params-cm`
belongs to the install manifest fetched from a URL, not to this repo. Skip it and Argo
appears broken in ways that don't point at the cause — redirect loops, or TLS errors from
the gateway rather than from Argo.

> `--insecure` means "don't do TLS yourself", not "no encryption". TLS terminates once,
> at the edge, exactly as it does for `web`. The gateway→Argo hop is plaintext inside the
> cluster; the `argocd` namespace has no Istio sidecar (deliberately — Argo's gRPC and
> Redis traffic don't take injection well), so there's no mTLS on that hop either. Worth
> revisiting under Phase 8.

Then the app-of-apps root — **the last hand-applied manifest in this repo**:

```bash
kubectl apply -f root.yaml
```

It syncs `argocd/`, which creates the `infra`, `staging`, and `prod` Applications, which
in turn adopt everything from step 6. Adoption causes no restarts: Argo diffs git against
live, finds them equivalent, and just stamps its tracking annotation.

`infra` is a **manual** app, so the Argo ingress won't exist until you sync it. Chicken
and egg — you can't reach the UI to click Sync. Trigger it through the Kubernetes API
instead:

```bash
kubectl patch app infra -n argocd --type merge -p '{"operation":{"sync":{}}}'
kubectl get gw,virtualservice -n argocd     # note: `gw`, see below
```

> Anything the Argo CLI does, it does by mutating the `Application` CRD. So `kubectl`
> still works when Argo's own ingress doesn't — which is exactly when you need it.

> **`kubectl get gateway` lies here.** Two API groups register `gateways`:
> `gateway.networking.k8s.io` (the Gateway API CRD Istio installs) and
> `networking.istio.io`. kubectl silently picks the former and reports nothing found.
> Use `gw` or `gateways.networking.istio.io`.

Finally, secure the admin account:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
argocd login argocd.rcamine.com --grpc-web

argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

Change the password **before** deleting the Secret — deleting it doesn't invalidate the
old password, it only removes your ability to look it up. Don't pass `--password` on the
command line; it lands in shell history.

> **`--grpc-web` is required.** The CLI speaks gRPC, which needs HTTP/2 end to end. Istio
> forwards to the Service port named `http` and treats it as HTTP/1.1, so native gRPC
> fails with opaque transport errors. `--grpc-web` tunnels gRPC over HTTP/1.1.

> **If you must use a port-forward** (before the ingress exists, or to debug it), note
> that `server.insecure` makes Argo serve **plain HTTP** — so it's
> `argocd login localhost:8443 --plaintext`, not `--insecure`. Those flags are not
> synonyms: `--insecure` means "do TLS but skip verification" and fails against a
> plaintext server with `server gave HTTP response to HTTPS client`. Also remember a
> port-forward binds to one pod — restarting `argocd-server` silently breaks it.

> **Why manual:** the bootstrap paradox. Something has to create the thing that creates
> everything else. `root.yaml` reduces that to exactly one command, forever.

---

## 8. GHCR package visibility — nothing to do (if the repo is public)

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

## 9. Local DNS for browser testing

The cluster isn't reachable from the public internet, so point the hostnames at the
local ingress:

```
# /etc/hosts
127.0.0.1  staging.rcamine.com
127.0.0.1  argocd.rcamine.com
127.0.0.1  rcamine.com
```

`curl --resolve staging.rcamine.com:443:127.0.0.1 https://...` does the same thing
without touching `/etc/hosts` — handy for scripted checks, but the browser needs the
real entries.

> **Why manual:** the real DNS records intentionally don't point at a laptop.

---

## Still to come

- **prod on the apex `rcamine.com`** — prod has no external ingress at all today; it's
  reachable only in-cluster at `http://web.prod`. The wildcard cert already covers the
  apex, so it needs a Gateway/VirtualService mirroring staging's and a `/etc/hosts` entry.
- **A NetworkPolicy for `argocd`** — `staging` and `prod` are default-deny, but `argocd`
  has no policy at all, and it now hosts an admin console reachable through the gateway.
  It isn't a copy of the existing pair: Argo needs egress to GitHub and the Kubernetes
  API, and ingress from `istio-system`.
