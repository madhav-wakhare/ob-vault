---

title: Migrating a Live Helm Release to Argo CD (eso-config)

tags: [kubernetes, argocd, helm, gitops, sre, external-secrets]

created: 2026-08-04

---
# Migrating a Live Helm Release to Argo CD — Without Downtime

A hands-on walkthrough of moving one Helm release (`eso-config`) onto Argo CD on a
running cluster, with nothing going down. Written as a guide I can follow again.

The object being migrated was **live and in use** the whole time. Two workloads
depended on it. It never stopped working.

---

## 1. The Starting Point

### What was on the cluster

A 4-node Minikube cluster with everything installed by Helm:

| Helm release | Namespace | What it is |
| --- | --- | --- |
| `external-secrets` | `eso-ns` | External Secrets Operator (ESO) + its CRDs |
| **`eso-config`** | `eso-ns` | **← the one we migrated** |
| `vault` | `vault` | HashiCorp Vault (the secret backend) |
| `postgres-db` | `student-api` | PostgreSQL |
| `student-api` | `student-api` | The app |

`eso-config` is a tiny chart. It manages **exactly one object**:

```
ClusterSecretStore/vault-backend
```

That object tells ESO: *"to fetch secrets, log into Vault at this address, using
this Kubernetes ServiceAccount, as this Vault role."*

### Why it was the right first migration

Pick your first migration for **boringness**, not importance:

- ✅ One object only
- ✅ No `randAlphaNum` / `lookup` (generated values — see §9)
- ✅ No Helm hooks
- ✅ No immutable fields
- ✅ No PersistentVolumeClaims
- ✅ No pods → nothing restarts
- ✅ Chart in git already matched the cluster exactly (zero drift)

> [!tip] Rule of thumb
> If your easiest release is hard to migrate, every other release will be worse.
> Start with the one that has the least state.

### Two details that made it interesting

**It's cluster-scoped.** The Helm release lives *in* `eso-ns`, but a `ClusterSecretStore` belongs to no namespace. That mismatch causes real gotchas
(§9.2).

**Two live consumers depended on it:**

```
NAMESPACE NAME STORE STATUS READY
student-api postgres-db vault-backend SecretSynced True
student-api student-api-db-url vault-backend SecretSynced True
```

> [!warning] The failure mode here is silent
> If the store disappears, those ExternalSecrets stop refreshing — but the
> Kubernetes Secrets they already created **stay in place**. Nothing crashes.
> You find out weeks later when a password rotates.
>
> A silent failure is worse than a loud one. This is why the plan never deletes
> and recreates the object.

---

## 2. Concept: Where Helm Ownership Actually Lives

Most people think "Helm owns this" is one thing. It's **four separate records**,
and nothing links them. Deleting one does not clean up the others.

| # | Record | Where it lives | What it does | Who reads it |
| --- | --- | --- | --- | --- |
| 1 | **Release ledger** ("the receipt") | `Secret sh.helm.release.v1.<release>.v<N>` in the release namespace | gzipped JSON: rendered manifest, values, chart, status | `helm list`, `upgrade`, `rollback`, `uninstall` |
| 2 | **Ownership annotations** ("the sticker") | on the object: `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace` | Helm's collision check — refuses to touch a foreign object | `helm upgrade` |
| 3 | **Managed-by label** | on the object: `app.kubernetes.io/managed-by: Helm` | Advisory only | humans |
| 4 | **`managedFields`** | on the object, maintained by the **API server** | per-field ownership, per manager | the API server, on every write |
## What SSA is, in one sentence

> Instead of your tool figuring out what changed, you send the API server the fields **you care about**, and the API server keeps a record of **who owns which field**.

That record is **`managedFields`**

### The asymmetry that matters most

Decode the release ledger and compare it to the live object:

```bash
kubectl get secret sh.helm.release.v1.eso-config.v1 -n eso-ns \
-o jsonpath='{.data.release}' | base64 -d | base64 -d | gzip -d \
> /tmp/before-release.json
```

The **rendered manifest inside the receipt** contains only what the chart
template produced:

```yaml
labels:
app.kubernetes.io/name: eso-config
app.kubernetes.io/instance: eso-config
```

But the **live object** had four metadata fields:

```yaml
annotations:
meta.helm.sh/release-name: eso-config # ← not in the chart
meta.helm.sh/release-namespace: eso-ns # ← not in the chart
labels:
app.kubernetes.io/instance: eso-config # from the chart
app.kubernetes.io/managed-by: Helm # ← not in the chart
app.kubernetes.io/name: eso-config # from the chart
```

**Helm injects three fields at apply time that the chart never rendered.**

| Field | Comes from | After migration |
| --- | --- | --- |
| `app.kubernetes.io/name`, `.../instance` | **the chart** | Argo CD renders them too → stays |
| `app.kubernetes.io/managed-by: Helm` | **Helm injects** | nothing renders it → **remove by hand** |
| `meta.helm.sh/release-*` | **Helm injects** | nothing renders it → **remove by hand** |

> [!important] Why this matters
> Argo CD renders the same chart, so its desired state **legitimately lacks**
> those three fields. It will never clean them up on its own. Knowing which
> fields the chart owns vs. which Helm injected is the difference between a
> clean finish and mystery leftovers.

---

## 3. Concept: Server-Side Apply and `managedFields`

### The version check that changes everything

```bash
helm version # → v4.2.3
```

Then in the decoded receipt:

```
apply_method: ssa
```

And in the object's `managedFields`:

```
manager: helm, operation: Apply
```

**Helm 4 applies server-side.** This matters because almost every Helm→Argo CD
guide online assumes **Helm 3**, where ownership is a client-side annotation
called `kubectl.kubernetes.io/last-applied-configuration` and adoption is a
three-way-merge problem.

With Helm 4 there is **no such annotation**. Instead the **API server** tracks
who owns which field.

### How to read `managedFields`

```bash
kubectl get clustersecretstore vault-backend --show-managed-fields \
-o jsonpath='{.metadata.managedFields}' | python3 -m json.tool
```

Each entry is one manager's claim. `fieldsV1` is a tree mirroring the object:

```
"f:spec": { "f:provider": { "f:vault": { "f:path": {} } } }
→ this manager owns spec.provider.vault.path
```

- `f:` means "field"
- empty `{}` marks a leaf this manager owns
- `operation: Apply` = server-side apply
- `operation: Update` = imperative write (`kubectl edit`, a controller patch)
- `subresource: status` = a **separate write path**

### The SSA rule that makes zero-downtime adoption possible

> [!important] The whole migration hinges on this
> When a **second** manager applies a field the **first** already owns:
> - **Same value** → ownership becomes **shared**. Both managers listed. No error.
> - **Different value** → **`409 Conflict`**, naming the conflicting manager and
> fields, unless the applier forces.
>
> And: **a field you don't specify is not a field you delete** — it's a field you
> don't claim. The other manager keeps it.

That second half is why Helm's annotations survived, and why the `spec` could not
possibly change.

### A script worth keeping

Save as `owners.py`. This is the single most useful diagnostic for any ownership
question:

```python
import json, sys

obj = json.load(sys.stdin)
own = {}

def walk(d, path, mgr):
for k, v in d.items():
if k in (".", "f:."):
continue
p = path + "." + (k[2:] if k.startswith("f:") else k)
if isinstance(v, dict) and v:
walk(v, p, mgr)
else:
own.setdefault(p.lstrip("."), set()).add(mgr)

for e in obj["metadata"]["managedFields"]:
m = e["manager"] + ("/status" if e.get("subresource") else "")
walk(e.get("fieldsV1", {}), "", m)

w = max(len(k) for k in own)
for k in sorted(own):
o = sorted(own[k])
tag = " <== SHARED" if len(o) > 1 else ""
print(k.ljust(w) + " " + ", ".join(o) + tag)
```

Run it:

```bash
kubectl get <kind> <name> --show-managed-fields -o json | python3 owners.py
```

---

## 4. Concept: Argo CD Does NOT Use Helm to Deploy

This is the biggest mental-model correction.

**Argo CD uses Helm only as a text renderer.** It runs `helm template`, gets a
string of YAML back, and that's the end of Helm's involvement.

| | `helm template` | `helm install` / `upgrade` |
| --- | --- | --- |
| Renders YAML | ✅ | ✅ |
| Talks to the cluster | ❌ never | ✅ |
| Creates a release ledger | ❌ | ✅ |
| Injects `meta.helm.sh/*` | ❌ | ✅ |
| Applies to the API server | ❌ | ✅ as manager `helm` |
| Runs hooks | ❌ | ✅ |

**Argo CD only ever uses the left column.** The repo-server (which renders) is
stateless and has *no credentials for your target cluster* — it physically cannot
apply anything. The **application-controller** applies the YAML with its own
client as field manager `argocd-controller`.

> Think of `helm template` as a compiler and Argo CD as the thing that runs the
> output. Helm is a build-time dependency, not a deployment tool.

### How to prove it

Sync 100 times and then:

```bash
helm history eso-config -n eso-ns
```

Revision stays at **1**. No new `sh.helm.release.v1.eso-config.v2` secret ever
appears. If Argo CD were running `helm upgrade`, you'd get a new revision every
sync.

### What this breaks (know before you migrate other charts)

| Helm feature | Under Argo CD |
| --- | --- |
| `lookup` function | **Always returns nil** — repo-server has no cluster access. The idiom *"read existing Secret, else generate"* silently always generates. |
| `randAlphaNum` | **Regenerates on every render** → permanently OutOfSync → with self-heal, an endless apply loop |
| Helm hooks | Argo CD reads `helm.sh/hook` from the YAML and maps it to **its own** sync phases. A hook that ran once per `helm upgrade` now runs on **every sync**. |
| `helm rollback` | Meaningless. Rollback = `git revert` |
| `--wait`, `--atomic`, `--timeout` | Replaced by sync waves, health checks, retry policy |

---

## 5. Concept: Argo CD Resource Tracking (the landmine)

Argo CD must answer *"which Application owns this object?"* for everything in the
cluster. It has two ways.

### `label` — the DEFAULT, and it was going to break

Writes `app.kubernetes.io/instance: <app-name>` onto the object.

**Problem:** the `eso-config` chart template already writes that exact field:

```yaml
# helm/eso-config/templates/clustersecretstore.yaml
labels:
app.kubernetes.io/instance: {{ .Release.Name }} # → "eso-config"
```

Two systems, one field. They'd agree **only by the coincidence** that the Helm
release and the Argo app share the name `eso-config`.

Other problems with label tracking:
- Labels are capped at **63 characters** — long app names get silently truncated
- Carries no kind/namespace, so two same-named apps are indistinguishable
- **Worst case:** a *different* Application that happens to render the same label
silently steals the resource. No error, no event.

### `annotation` — what to use

Writes a full identity string:

```
argocd.argoproj.io/tracking-id: eso-config:external-secrets.io/ClusterSecretStore:eso-ns/vault-backend
└─ app ─┘ └──── group/kind ────┘ └── ns/name ──┘
```

Cannot collide with chart labels. No length limit. Encodes full identity.

> [!danger] Set this BEFORE Argo CD manages anything
> Switching tracking method later un-tracks every already-tracked resource for a
> moment. On a cluster with `prune: true` anywhere, **that is an outage**.

### Surprise finding

The tracking ID uses the Application's `destination.namespace` (`eso-ns`) **even
for a cluster-scoped resource**. I expected an empty segment; it isn't.

**Consequence:** change `destination.namespace` and the tracking ID changes →
Argo CD no longer recognises the resource as its own. With `prune: true`, that's
how a "harmless" namespace edit orphans a resource and creates a duplicate.

---

## 6. Concept: Ownership vs Reconciliation vs Adoption

Three words people use interchangeably. They're different:

**Ownership** — *who is recorded as responsible.*
Helm: receipt + `meta.helm.sh/*`. Argo CD: tracking annotation. Kubernetes:
`managedFields`, per field, per manager.

**Reconciliation** — *the act of comparing desired to live and closing the gap.*
Argo CD does this **continuously**. Helm doesn't reconcile at all — it acts once
per `helm upgrade` and then stops.

**Adoption** — *transferring ownership of an object that already exists.*

> [!important] Why Helm and Argo CD can safely overlap
> **Argo CD is continuous. Helm is episodic.**
>
> A Helm release nobody runs `helm upgrade` on exerts **zero force** on the
> cluster. That's what makes the overlap window safe — and exactly why an
> accidental `helm upgrade` during it is destructive.
>
> The overlap is safe because nobody typed a command, not because the design
> protects you.

---

## 7. What I Actually Did

### Step 0 — Install Argo CD (via Helm, manually)

`argocd-values.yaml`:

```yaml
global:
nodeSelector:
type: dependent_services # pin to the right node

crds:
install: true
keep: true # see §9.5 — this one matters

configs:
cm:
application.resourceTrackingMethod: annotation # ← THE critical setting (§5)
timeout.reconciliation: 60s # poll git every 60s (default 3m)
params:
server.insecure: true # for port-forward
controller.diff.server.side: true # ← see §9.1

dex:
enabled: false
notifications:
enabled: false

server:
service:
type: ClusterIP
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
--namespace argocd --create-namespace \
--version 10.2.2 \
-f argocd-values.yaml \
--wait --timeout 10m
```

**Verify before continuing:**

```bash
kubectl get pods -n argocd -o wide
kubectl get cm argocd-cm -n argocd \
-o jsonpath='{.data.application\.resourceTrackingMethod}{"\n"}' # → annotation
kubectl get cm argocd-cmd-params-cm -n argocd \
-o jsonpath='{.data.controller\.diff\.server\.side}{"\n"}' # → true
kubectl get crd | grep argoproj
```

> [!note] Argo CD installed by Helm is itself a Helm release
> `helm list -A` now shows a 6th release. That's fine — something has to deploy
> the deployer. But it means Argo CD is in the same ownership situation we just
> studied, and could be migrated the same way later.

---

### Step 1 — Recon (read-only). Capture a baseline

**Why:** the acceptance criterion for this migration is *"the object's `spec` is
byte-identical before and after."* If you can't prove that, a reviewer's only
option is to trust you — and migrations get rolled back out of doubt far more
often than out of failure.

```bash
mkdir -p /tmp/eso-migration

# The spec — the ONLY thing that must not change
kubectl get clustersecretstore vault-backend -o jsonpath='{.spec}' \
| python3 -m json.tool > /tmp/eso-migration/before-spec.json

# Field ownership, as the API server sees it
kubectl get clustersecretstore vault-backend --show-managed-fields \
-o jsonpath='{.metadata.managedFields}' | python3 -m json.tool \
> /tmp/eso-migration/before-managedfields.json

# Helm's receipt, decoded (readable)
kubectl get secret sh.helm.release.v1.eso-config.v1 -n eso-ns \
-o jsonpath='{.data.release}' | base64 -d | base64 -d | gzip -d \
> /tmp/eso-migration/before-release.json

# Helm's receipt, raw — this is the UNDO BUTTON
kubectl get secret sh.helm.release.v1.eso-config.v1 -n eso-ns -o yaml \
> /tmp/eso-migration/helm-release-backup.yaml
```

> [!tip] Diff the `spec`, never `-o yaml`
> The full object carries `resourceVersion`, `uid`, `generation`,
> `creationTimestamp` and `status` — all change for innocent reasons. Diffing
> them generates noise and teaches you nothing.

**The go/no-go check:**

```bash
helm template eso-config helm/eso-config -n eso-ns | kubectl diff -f -
echo $? # bash. In fish it's: echo $status
```

Result: **exit 0** — zero drift. Git and cluster agreed exactly.

> [!danger] If this is NOT zero, STOP
> It means the chart in git no longer matches what Helm deployed — you have
> **pre-existing drift**. Migrating now means adopting an unknown state.
> Fix it with Helm, in Helm's world, first.

---

### Step 2 — Create the Application (observe only, don't sync)

**Prerequisite:** push the branch. Argo CD's repo-server clones **GitHub**, not
your laptop. A file you haven't pushed does not exist as far as the cluster is
concerned.

```bash
git push -u origin argocd-migration
git ls-tree origin/argocd-migration --name-only helm/eso-config/
```

I used the UI (**Applications → + NEW APP**), but the wizard has two traps:

> [!warning] Trap 1 — no Release Name field
> The 3.4 wizard's HELM panel offers only VALUES FILES, VALUES and PARAMETERS.
> `spec.source.helm.releaseName` **isn't in the form.** Use the **EDIT AS YAML**
> toggle near the CREATE button.
>
> (It defaults to the Application name, so if those match you're fine anyway.)

> [!warning] Trap 2 — the PARAMETERS fields
> They look pre-filled with your chart values. They're actually **empty
> overrides** displaying current values as placeholders. Type in any of them and
> Argo CD writes it to `spec.source.helm.parameters` — config that lives in the
> cluster but **not in git**, overriding `values.yaml`, invisible unless you read
> the Application spec. **Leave them all alone.**

The YAML I used:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
name: eso-config
namespace: argocd
spec:
project: default
source:
repoURL: https://github.com/madhav-wakhare/SRE-Bootcamp.git
targetRevision: argocd-migration
path: helm/eso-config
helm:
releaseName: eso-config
valueFiles:
- values.yaml
destination:
server: https://kubernetes.default.svc
namespace: eso-ns
syncPolicy:
syncOptions:
- ServerSideApply=true
```

**Why each choice:**

| Setting | Reason |
| --- | --- |
| `ServerSideApply=true` | Helm 4 already owns these fields via SSA. Both writers speaking the same protocol makes `managedFields` readable — and it's the production-correct choice. |
| `releaseName: eso-config` | Argo CD sets `.Release.Name` from this. The chart renders it into a label. Pin it so a future app rename can't silently rewrite a label on a live cluster-scoped object. |
| `namespace: eso-ns` | A ClusterSecretStore is cluster-scoped — this does **not** place it anywhere. It only sets `.Release.Namespace` for the render. Matched to the Helm release namespace so both systems agree. |
| **no `automated:` block** | Sync stays manual. The whole point of this step is seeing what Argo CD *would* do before it does it. |
| **no `resources-finalizer`** | See §9.4. This is the rollback. |
| **no `helm.parameters`** | `values.yaml` in git is the only source of values. |

**Result: `OutOfSync / Healthy`.**

### Understanding that OutOfSync

Not drift. Not Helm. **Argo CD simply hadn't claimed the resource yet.**

Proof — reproduce Argo CD's comparison locally:

```bash
# A) plain chart render vs live
helm template eso-config helm/eso-config -n eso-ns | kubectl diff -f -
# → exit 0, ZERO diff

# B) same render + the tracking annotation
# → exactly ONE added line:
```

```diff
annotations:
+ argocd.argoproj.io/tracking-id: eso-config:external-secrets.io/ClusterSecretStore:eso-ns/vault-backend
meta.helm.sh/release-name: eso-config
meta.helm.sh/release-namespace: eso-ns
```

**One missing annotation → OutOfSync.** Read it as: *"Everything about this
object already matches git. The only thing I'd change is writing my own name
on it."*

Two things to notice in that diff:

1. **`meta.helm.sh/*` survives the merge**, even though it's nowhere in git.
That's SSA doing its job — fields owned by `helm` and unspecified by the
applier are left alone. **This is the mechanical proof that adoption will be
non-destructive.**
2. **Had this diff contained a `spec` change, stop.** A one-line annotation diff
is the green light.

> [!important] `Synced` ≠ "Argo CD owns this"
> `Synced` means *"applying would change nothing."* Those are very different
> claims. Always check the tracking annotation explicitly.

---

### Step 3 — The adoption sync

UI: app → **SYNC** → leave PRUNE and DRY RUN unchecked → **SYNCHRONIZE**.

**What happens on the wire:**

1. Controller fetches the render from repo-server
2. Injects the tracking annotation into the desired manifest
3. Applies as `PATCH`, content-type `application/apply-patch+yaml`,
`fieldManager=argocd-controller`
4. API server routes the write through **ESO's `secretstore-validate` webhook**
(`failurePolicy: Fail`, cluster scope, on CREATE/UPDATE/DELETE). If ESO's
webhook pod were down, this sync would fail.
5. API server merges by field ownership and rewrites `managedFields`

**Verify — four questions:**

```bash
# Q1. Did Argo CD claim it?
kubectl get clustersecretstore vault-backend \
-o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'

# Q2. THE acceptance criterion
kubectl get clustersecretstore vault-backend -o jsonpath='{.spec}' \
| python3 -m json.tool > /tmp/eso-migration/after-spec.json
diff /tmp/eso-migration/before-spec.json /tmp/eso-migration/after-spec.json \
&& echo "SPEC UNCHANGED"

# Q3. Who owns what now?
kubectl get clustersecretstore vault-backend --show-managed-fields -o json \
| python3 owners.py

# Q4. Did anything downstream notice?
kubectl get clustersecretstore vault-backend
kubectl get externalsecrets -A
helm list -n eso-ns
```

**Q3 output — the money shot:**

```
metadata.annotations.argocd.argoproj.io/tracking-id argocd-controller
metadata.annotations.meta.helm.sh/release-name helm
metadata.annotations.meta.helm.sh/release-namespace helm
metadata.labels.app.kubernetes.io/managed-by helm
metadata.labels.app.kubernetes.io/instance argocd-controller, helm <== SHARED
metadata.labels.app.kubernetes.io/name argocd-controller, helm <== SHARED
spec.provider.vault.server argocd-controller, helm <== SHARED
spec.provider.vault.path argocd-controller, helm <== SHARED
spec.provider.vault.version argocd-controller, helm <== SHARED
spec.provider.vault.auth.kubernetes.mountPath argocd-controller, helm <== SHARED
spec.provider.vault.auth.kubernetes.role argocd-controller, helm <== SHARED
spec.provider.vault.auth.kubernetes.serviceAccountRef.name argocd-controller, helm <== SHARED
spec.provider.vault.auth.kubernetes.serviceAccountRef.namespace argocd-controller, helm <== SHARED
status.capabilities external-secrets/status
status.conditions external-secrets/status
```

**Three managers, four groups of fields:**

1. **Nine SHARED fields** — everything the chart renders. Both `helm` and
`argocd-controller` own the same field. No conflict, because both applied
*identical values*. The SSA rule from §3, confirmed live.
2. **Three `helm`-only fields** — Helm's injected metadata. Argo CD never claimed
them because nothing in git renders them. **This is the residue to clean up.**
3. **One `argocd-controller`-only field** — the tracking annotation.
4. **`status`, owned by `external-secrets` on a subresource** — untouched by both.
It will never show as drift, because a subresource is a different write path
Argo CD isn't in. (On other resources you'd need `ignoreDifferences` for this.)

**And the critical result: `SPEC UNCHANGED`.** Store still `Ready=True`. Both
ExternalSecrets still `SecretSynced`. `helm list` still showed `eso-config` rev 1
`deployed` — **Helm had no idea anything happened**, so `helm rollback` was still
a working escape hatch.

> [!success] How ownership transferred with zero downtime
> Because Argo CD's desired values were **identical** to what was already live,
> the server-side merge was a **no-op everywhere except one new annotation**.
> The API server never rewrote `spec`. ESO never saw a change. The
> ExternalSecrets never lost their store.
>
> **Adoption was a metadata change and nothing else.** That's the whole trick:
> make git match reality *first*, then let Argo CD claim it.

---

### Step 4 — Retire Helm (order matters)

Two systems now claimed the object. Fine as a transition, unacceptable as a
steady state: the next person runs `helm upgrade` and you have two writers with
different desired states and no arbiter.

#### Why receipt-first, sticker-second

**Take away the ability to act before you take away the warning label.**

| Order | What happens if someone runs `helm upgrade` in the gap |
| --- | --- |
| **Receipt first** ✅ | Helm has forgotten the release → treats it as a new install → hits the sticker → **refuses, changes nothing.** The sticker is your safety net during the gap. |
| **Sticker first** ❌ | Helm still remembers the release and still believes it owns the object. No safety net — you removed it. |

> [!danger] Never run `helm uninstall`
> `helm uninstall eso-config` doesn't just delete the receipt — **it deletes the
> ClusterSecretStore itself**, breaking both ExternalSecrets silently.
>
> Helm 4 has `--keep-history` (keeps the receipt, **still deletes the
> resources**). There is **no** `--keep-resources` flag. There is no safe version
> of `helm uninstall` here.
>
> Delete the Secret directly. It does exactly one thing, and you can see what
> that thing is.

#### The commands

```bash
# 1. Back up the receipt — your undo button
kubectl get secret sh.helm.release.v1.eso-config.v1 -n eso-ns -o yaml \
> /tmp/eso-migration/helm-release-backup.yaml

# 2. Check how many receipts exist (one per revision!)
kubectl get secret -n eso-ns -l owner=helm,name=eso-config

# 3. Confirm Argo CD really is in charge BEFORE removing the safety net
kubectl get app eso-config -n argocd # Synced / Healthy
kubectl get clustersecretstore vault-backend \
-o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'

# 4. Delete the receipt
kubectl delete secret sh.helm.release.v1.eso-config.v1 -n eso-ns

# 5. Helm should now only know about external-secrets
helm list -n eso-ns

# 6. Remove the sticker (trailing "-" means "delete this field")
kubectl annotate clustersecretstore vault-backend \
meta.helm.sh/release-name- meta.helm.sh/release-namespace-
kubectl label clustersecretstore vault-backend app.kubernetes.io/managed-by-
```

> [!warning] Do NOT delete `app.kubernetes.io/instance` or `.../name`
> The chart renders those. Argo CD renders the same chart, so it puts them
> straight back within a minute and you'll think something is haunted.
>
> Only the three fields in step 6 were Helm's own additions — exactly the three
> the ownership table showed as `helm`-only.

#### Final state

```
labels → {"app.kubernetes.io/instance":"eso-config","app.kubernetes.io/name":"eso-config"}
annotations → {"argocd.argoproj.io/tracking-id":"eso-config:...:eso-ns/vault-backend"}
store → Valid / ReadWrite / Ready=True
both ES → SecretSynced
app → Synced / Healthy
helm list → external-secrets only
spec → IDENTICAL to the original baseline
```

> [!note] Removing `managed-by` did not cause drift
> That label was never in git, so Argo CD has no opinion about it. If the app had
> gone OutOfSync here, it would mean we deleted something the chart renders.

#### One leftover that stays, and is fine

Run the ownership table again and `helm` is **still listed** as an owner of the
nine shared fields.

That's normal. `managedFields` is the **API server's** record of who once wrote
what. Deleting Helm's receipt doesn't touch it. It's now just a name with no
program behind it — nothing will ever write as `helm` again.

**Helm 3 wouldn't leave this behind at all.** Helm 4 applies server-side, so it
registers with the API server, and that registration outlives the release.

#### If you need to undo

```bash
kubectl apply -f /tmp/eso-migration/helm-release-backup.yaml
kubectl annotate clustersecretstore vault-backend \
meta.helm.sh/release-name=eso-config meta.helm.sh/release-namespace=eso-ns
kubectl label clustersecretstore vault-backend app.kubernetes.io/managed-by=Helm
```

`helm list` shows the release again and `helm rollback` works. **Test this path
before you need it.**

---

## 8. Step 5 — Make it self-correcting (next up, not yet done)

### The two switches

| | Watches | Notices in |
| --- | --- | --- |
| **Auto-sync** | git | ~60s (the poll interval) |
| **Self-heal** | the live object | ~seconds (a watch event) |

Self-heal is what turns drift from *"something you discover"* into *"something
that fixed itself."*

UI: app → **DETAILS** → **ENABLE AUTO-SYNC**, then **SELF HEAL**. Leave PRUNE off.

### Prove it works

```bash
kubectl patch clustersecretstore vault-backend --type merge \
-p '{"spec":{"provider":{"vault":{"path":"wrong-path"}}}}'

kubectl get clustersecretstore vault-backend -o jsonpath='{.spec.provider.vault.path}{"\n"}'
sleep 15
kubectl get clustersecretstore vault-backend -o jsonpath='{.spec.provider.vault.path}{"\n"}'

kubectl get app eso-config -n argocd \
-o jsonpath='{.status.operationState.operation.initiatedBy}{"\n"}'
```

**Time it.** Heals in ~2s → self-heal's watch caught it. Closer to a minute →
the git poll caught it, which means self-heal isn't actually on.

### Two things about self-heal

**It backs off.** Argo CD doesn't retry instantly forever. If something else
keeps rewriting the same field, the fight slows down rather than melting the API
server.

**So an app syncing every few seconds non-stop is not self-heal misbehaving** —
it means two things are writing the same field. **The fix is never to turn
self-heal off. It's to find the other writer.**

### Why prune goes last

Prune deletes live resources Argo CD tracks but git no longer produces.

The danger isn't prune, it's an **empty render**. A bad values edit or a broken
branch makes the chart render nothing — and *"git produces nothing"* looks exactly
like *"delete everything."*

**Prune is the only irreversible switch in an Application.** Enable it last,
after the render has been stable for a while.

### Recommended order (one commit each)

```
1. adopt (manual sync)
2. automate (auto-sync + selfHeal, prune: false)
3. prune (prune: true)
4. finalize (add resources-finalizer — only after Helm is fully gone)
```

Four changes, four verifications. When something breaks you know which one did it.

---

## 9. Production Scenarios & Gotchas

### 9.1 Server-side diff — turn it on

```yaml
configs:
params:
controller.diff.server.side: true
```

**Without it:** Argo CD does a client-side three-way merge. Fields the *API
server* defaults (e.g. `deletionPolicy` on an ExternalSecret, `volumeMode` in a
`volumeClaimTemplate`) look like drift, and apps never leave OutOfSync.

**With it:** Argo CD sends the desired state as an SSA **dry-run** and diffs the
API server's merged answer. The API server — not Argo CD — decides what the merge
would be, so defaulted fields and other managers' fields don't appear as drift.

Verify it's actually wired in:

```bash
kubectl get statefulset argocd-application-controller -n argocd \
-o jsonpath='{.spec.template.spec.containers[0].env[*].name}' \
| tr ' ' '\n' | grep -i diff
# → ARGOCD_APPLICATION_CONTROLLER_SERVER_SIDE_DIFF
```

### 9.2 Cluster-scoped resources need an empty-namespace destination

We used `project: default`, which permits everything. In production you'd write a
restrictive `AppProject` — and here's the gotcha that catches everyone:

```yaml
destinations:
- server: https://kubernetes.default.svc
namespace: eso-ns
- server: https://kubernetes.default.svc
namespace: "" # ← REQUIRED for cluster-scoped resources
clusterResourceWhitelist:
- group: external-secrets.io
kind: ClusterSecretStore
```

**Kind and destination are two separate checks. Both must pass.** Whitelisting
the kind is not enough — without the `""` destination, sync fails with
`ClusterSecretStore "vault-backend" is not permitted in project`.

And a project violation is **not a failed apply** — the controller refuses to
submit the object at all and reports `ComparisonError`. The guardrail fires
*before* any API call, so a bad commit can't half-apply.

> [!tip] Never use `clusterResourceWhitelist: '*'` in production
> Cluster-scoped objects are the blast radius. A namespaced mistake is contained
> by its namespace; a `ClusterRoleBinding` mistake is not. Enumerate them, with a
> comment saying which chart needs each one.

### 9.3 Generated values — the permanent reconciliation loop

Because Argo CD uses `helm template`:

```yaml
# ANY of these in a chart = trouble under Argo CD
annotations:
nonce: {{ randAlphaNum 8 }} # regenerates every render
password: {{ randAlphaNum 32 }} # regenerates every render
{{- $existing := lookup "v1" "Secret" .Release.Namespace "my-secret" }} # always nil
```

**Symptom:** app never reaches Synced. With self-heal on, it applies every few
seconds forever, hammering the API server. At 500 Applications this is an
incident.

**Grep every chart before migrating it:**

```bash
grep -rn "randAlphaNum\|randAlpha\|derivePassword\|uuidv4\|genCA\|genSelfSigned\|lookup " helm/
```

**Fixes, best to worst:**
1. Don't generate secrets in charts at all — that's what ESO/Vault is for
2. Let the *workload* generate it at runtime, and `ignoreDifferences` the field
3. Pre-create the Secret out-of-band and reference it

### 9.4 The `resources-finalizer` is a loaded gun during migration

```yaml
metadata:
finalizers:
- resources-finalizer.argocd.argoproj.io
```

This makes deleting the Application **cascade into deleting everything it
manages**.

**Mid-migration that is dangerous.** `kubectl delete app eso-config` — the
natural panic response — would delete the live ClusterSecretStore and silently
break both ExternalSecrets, while Helm's receipt still claimed the object existed.

**Without the finalizer**, deleting the Application just stops reconciliation and
leaves the resource running. **That is what a safe rollback looks like.** Add the
finalizer only at the very end, once Helm is gone and git is genuinely the only
source of truth.

Same reasoning applies to the root/app-of-apps pattern: an app-of-apps with a
finalizer turns one `kubectl delete` into a cluster wipe.

### 9.5 `crds.keep: true` — the difference between "removed Argo CD" and "removed the cluster"

Deleting the `Application` CRD deletes **every Application object** in the
cluster. Each one carrying a finalizer then cascades into its workloads.

Also: **`kubectl delete ns argocd` does not uninstall Argo CD** — the CRDs and
ClusterRoles are cluster-scoped and survive.

**Uninstall order matters:**
1. Strip Application finalizers **while the controller is still running** (only
Argo can clear an Argo finalizer)
2. *Then* remove the controller
3. Never delete the CRDs

Reverse that and the Applications wedge in `Terminating` forever.

### 9.6 Helm hooks become Argo hooks — with different timing

Argo CD reads `helm.sh/hook` annotations from the rendered YAML and maps them onto
its own phases:

| Helm | Argo CD |
| --- | --- |
| `pre-install`, `pre-upgrade` | `PreSync` |
| `post-install`, `post-upgrade` | `PostSync` |
| `helm.sh/hook-delete-policy: before-hook-creation` | `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation` |

**The dangerous part is the timing change.** A Helm hook runs once per
`helm upgrade` — a deliberate human action. An Argo hook runs **on every sync**,
whenever the controller decides to sync.

Think about a **database migration Job**. Under Helm it ran when you deployed.
Under Argo CD it may run on every reconcile. Idempotent hooks are fine;
non-idempotent ones are a data-integrity incident.

*(The Argo CD chart itself ships four `pre-install,pre-upgrade` resources —
`argocd-redis-secret-init` — so this is live in any Argo CD install.)*

### 9.7 Immutable fields — what will bite you on StatefulSets

`ClusterSecretStore` has none, which is another reason it was a good first
migration. But `postgres-db` has `volumeClaimTemplates`, which are **immutable
after creation**.

Changing them → the API server rejects the apply → sync fails.

`Replace=true` as a sync option "works" by doing `kubectl replace` — but for a
StatefulSet with bound PVCs that means **destroying and recreating**, i.e. data
loss. It's the tool of last resort, and never for a Secret whose data is written
at runtime.

Correct handling is a deliberate migration (new StatefulSet name, data copy, cut
over), not a sync option.

### 9.8 Renames are not free

Change `name: vault-backend` → `vault-backend-v2` in `values.yaml`:

- **With `prune: false`:** you now have **two** stores. The old one is
*orphaned*, the new one is unused until consumers point at it.
- **With `prune: true`:** the old one is deleted — and there's a window where the
name consumers reference **does not exist**.

Zero-downtime rename = create the new one, migrate consumers, *then* delete the
old one. Three commits, not one.

### 9.9 Missing vs Extraneous vs Orphaned

| State | Meaning | Argo CD's response |
| --- | --- | --- |
| **OutOfSync** | in git and live, contents differ | sync re-applies |
| **Missing** | in git, not live | sync creates |
| **Extraneous** | live, **tracked by this app**, not in git | pruned — only if `prune: true` |
| **Orphaned** | live in the app's namespace, tracked by **no** app | **warned about, never touched** |

The difference between the last two is **whether Argo CD's tracking annotation is
present**. An object Argo CD once created is *extraneous* — it knows it made it,
so pruning is defensible. An object it never created is *orphaned* — deleting it
would be destroying something it doesn't understand, so it never does.

Enable `orphanedResources: warn: true` on the project. **Mid-migration it will be
noisy, and the noise is correct** — it's an accurate statement that the namespace
has mixed ownership. As releases migrate, the list shrinks.

### 9.10 `targetRevision` should not be a branch

```yaml
targetRevision: argocd-migration # = "whatever someone pushed last"
targetRevision: v1.4.2 # = a release
```

A branch means your cluster changes whenever anyone pushes. Pin to a tag or a
commit SHA before production.

Related: **Argo CD reads git, not your working tree.** An uncommitted file does
not exist. Use `helm template ... | kubectl diff -f -` to check locally, and the
`argocd.argoproj.io/refresh: hard` annotation to skip the poll wait after pushing.

### 9.11 Set resource requests on Argo CD itself

The upstream chart ships **no defaults**, so all pods run as **BestEffort** QoS.
Under node pressure the kubelet evicts BestEffort pods **first** — and the pod you
least want evicted is the application-controller.

Fine on a lab node with 10 CPU. Wrong in production.

### 9.12 Admission webhooks are a hard runtime dependency

ESO registers `secretstore-validate` with `failurePolicy: Fail` on
CREATE/UPDATE/**DELETE** of `clustersecretstores`.

**If ESO's webhook pod is down you cannot sync, cannot diff, and cannot delete
this object.** That's a genuine dependency — but a *runtime availability* one, not
an ownership one. It never forces a migration order.

This is also why `Validate=true` (the default) is good: a structurally invalid
`vault.server` gets rejected at dry-run by the webhook, before the live object is
touched.

### 9.13 Don't hide the evidence with `ignoreDifferences`

There's a strong temptation to `ignoreDifferences` the Helm annotations so the app
goes green immediately. **Resist it.** Those annotations are the evidence the
migration isn't finished, and hiding them hides the one signal that tells you when
it is. Remove them (§7 Step 4); don't ignore them.

Use `ignoreDifferences` only for fields written by a **controller** rather than by
a human or by Helm. And pair it with:

```yaml
syncOptions:
- RespectIgnoreDifferences=true
```

Without that, Argo CD hides an ignored field from the *diff* and then applies it
anyway on the next *sync* — the field flaps, the app looks Synced, and the change
lands regardless.

---

## 10. Does anything have to migrate first?

For `eso-config`, **no** — and `external-secrets` should deliberately stay on Helm.

Four things could have forced the other order. None did:

| Possible dependency | Why it doesn't force an order |
| --- | --- |
| **The CRD** | Argo CD maps objects to REST endpoints via **API discovery**. It needs `clustersecretstores.external-secrets.io` to exist in the API server — it has **no opinion about who installed it**. |
| **The admission webhook** | A real dependency, but *runtime availability*, not ownership. And leaving ESO undisturbed is the best way to keep the webhook up. Migrating ESO first would mean restarting the very webhook that validates the object you're migrating — strictly more risk, for nothing. |
| **The `serviceAccountRef`** | A *data* reference resolved by ESO at fetch time. No `ownerReference`, no finalizer, no Helm relationship. |
| **Shared objects** | There are none. The `eso-config` receipt contains exactly one document. No object is claimed by both releases. |

> [!important] The general rule
> Helm ownership is **per-release and recorded per-object**. Two releases are
> entangled only if they **write the same object**.
>
> Runtime dependencies between releases — a CRD, a webhook, a service endpoint —
> constrain **when** you can sync. They never dictate **what order you must
> migrate**.

---

## 11. Reusable Checklist

Copy this per release.

### Before you touch anything

- [ ] `helm version` — is it 3 (client-side) or 4 (SSA)? Changes the whole story
- [ ] `helm get manifest <release> -n <ns>` — what does Helm think it deployed?
- [ ] `helm history <release> -n <ns>` — how many revisions (= how many receipts)?
- [ ] Grep the chart for `randAlphaNum`, `lookup`, `helm.sh/hook`
- [ ] List immutable fields in the chart (`volumeClaimTemplates`, `spec.selector`, `clusterIP`)
- [ ] Note every live consumer of the objects — what breaks, and how loudly?
- [ ] **`helm template <rel> <chart> -n <ns> | kubectl diff -f -` must exit 0.** If not, STOP and fix drift with Helm first
- [ ] Capture: `before-spec.json`, `before-managedfields.json`, `helm-release-backup.yaml`

### Argo CD config (once per cluster)

- [ ] `application.resourceTrackingMethod: annotation` — **before first sync**
- [ ] `controller.diff.server.side: true`
- [ ] `crds.keep: true`
- [ ] Resource requests/limits set (not BestEffort)
- [ ] Restrictive `AppProject`: scoped `sourceRepos`, enumerated
`clusterResourceWhitelist`, and a `namespace: ""` destination if any
resource is cluster-scoped

### Adoption

- [ ] Application created with **no `automated:`**, **no finalizer**, `ServerSideApply=true`
- [ ] `helm.releaseName` pinned to the existing Helm release name
- [ ] No `helm.parameters` in the spec (check for UI-injected overrides)
- [ ] Diff reviewed — **only the tracking annotation should differ**
- [ ] Sync
- [ ] `diff before-spec.json after-spec.json` is **empty**
- [ ] Tracking annotation present and correct
- [ ] `managedFields` reviewed — shared fields as expected, no conflicts
- [ ] Consumers still healthy
- [ ] `helm list` unchanged (rollback still available)

### Retiring Helm

- [ ] Receipt backed up somewhere durable (not `/tmp`)
- [ ] Argo CD confirmed `Synced / Healthy` **first**
- [ ] Receipt deleted (**never `helm uninstall`**)
- [ ] `helm list` no longer shows the release
- [ ] `meta.helm.sh/release-name`, `meta.helm.sh/release-namespace` removed
- [ ] `app.kubernetes.io/managed-by` removed
- [ ] Chart-rendered labels (`instance`, `name`) **left alone**
- [ ] App still `Synced` after cleanup
- [ ] Rollback path tested at least once

### Hardening (separate commit each)

- [ ] Auto-sync + `selfHeal: true`
- [ ] Drift test: `kubectl patch` a field, time the correction
- [ ] `prune: true` — only after the render has been stable
- [ ] `resources-finalizer` — last
- [ ] `targetRevision` pinned to a tag/SHA, not a branch
- [ ] Application YAML itself committed to git

---

## 12. Debug Cheat Sheet

### Which component do I look at?

| Component | Job | Look here when |
| --- | --- | --- |
| **repo-server** | clones git, runs `helm template`. **Stateless, no cluster access** | rendering fails, values wrong, `lookup` returns nil |
| **application-controller** | the reconcile loop: diff, apply, report. Owns the cluster cache | OutOfSync, sync errors, drift, pruning |
| **server** (API/UI) | read/write API in front of Application objects, RBAC, SSO | **UI/CLI problems only** — never the sync engine |
| **redis** | cache of renders and cluster state | stale diffs (do a hard refresh) |

### Commands

```bash
# App state
kubectl get app <app> -n argocd
kubectl get app <app> -n argocd -o jsonpath='{.status.resources}' | python3 -m json.tool
kubectl get app <app> -n argocd -o jsonpath='{.status.conditions}' | python3 -m json.tool
kubectl get app <app> -n argocd -o jsonpath='{.status.operationState}' | python3 -m json.tool
kubectl get app <app> -n argocd -o jsonpath='{.status.sync.revision}{"\n"}' # which commit

# Who triggered the last sync? (human vs automation — audit trail)
kubectl get app <app> -n argocd \
-o jsonpath='{.status.operationState.operation.initiatedBy}{"\n"}'

# Ownership
kubectl get <kind> <name> --show-managed-fields -o json | python3 owners.py

# Force an immediate git re-poll (no CLI needed)
kubectl patch app <app> -n argocd --type merge \
-p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Trigger a sync with kubectl only — this is exactly what `argocd app sync` does
kubectl patch app <app> -n argocd --type merge -p \
'{"operation":{"initiatedBy":{"username":"me"},"sync":{"syncStrategy":{"apply":{}},"dryRun":false,"prune":false}}}'

# Logs
kubectl logs -n argocd statefulset/argocd-application-controller --tail=200
kubectl logs -n argocd deploy/argocd-repo-server --tail=100
kubectl get events -n argocd --sort-by=.lastTimestamp | tail -20

# Local diff — what Argo CD would see
helm template <release> <chart> -n <ns> | kubectl diff -f -
echo $? # bash; fish uses: echo $status
```

### Symptom → cause

| Symptom | Likely cause |
| --- | --- |
| `OutOfSync`, diff is only the tracking annotation | Normal pre-adoption. Sync to adopt. |
| `OutOfSync` forever, tiny diff on defaulted fields | `controller.diff.server.side` is off |
| Syncs every few seconds, never Synced | Generated value (`randAlphaNum`) **or** a second writer |
| `... is not permitted in project` | `AppProject` missing the kind **or** the `namespace: ""` destination |
| `app path does not exist` | Branch/path not pushed to the remote |
| `metadata.annotations: Too long` | Client-side apply on a huge CRD — needs `ServerSideApply=true` |
| `409 Conflict` naming another manager | Two SSA managers, different values. **Read which fields before forcing.** |
| Deleted the Argo CD namespace, Argo CD still there | CRDs and ClusterRoles are cluster-scoped |
| Applications stuck `Terminating` | Finalizers, but the controller is already gone |
| App green, but Helm annotations still present | Adoption done, **cleanup not done** |

---

## 13. The Five Lessons

1. **Make git match reality first, then adopt.** A zero-diff baseline turns
adoption into a metadata change. That's what buys you zero downtime.

2. **Ownership isn't one thing.** Four separate records for Helm, plus Argo CD's
annotation, plus the API server's `managedFields`. Migration means dealing
with each, in order.

3. **`Synced` ≠ "owned."** `Synced` means *"applying would change nothing."*
Always verify the tracking annotation separately.

4. **Overlap is a feature.** Keeping Helm's receipt during adoption *is* the
rollback. Retire it last, in a separate step, once you've verified.

5. **One behavioural change per step.** Adopt → automate → prune → finalize.
When something breaks you know exactly which change did it.

> [!quote] The goal
> A good migration is **boring**. If it's exciting, something is wrong.