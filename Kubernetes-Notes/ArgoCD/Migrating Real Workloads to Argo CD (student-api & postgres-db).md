---

title: Migrating Real Workloads to Argo CD (student-api & postgres-db)

tags: [kubernetes, argocd, helm, gitops, sre, statefulset, postgres]

created: 2026-08-04

---
# Migrating Real Workloads to Argo CD

Part 2. [[Migrating a Live Helm Release to Argo CD (eso-config)|Part 1]] moved a single cluster-scoped config object. 
This one moves **things that can break**: an app that serves traffic, and a database with data on disk.

Both were migrated with **zero downtime, zero pod restarts, and zero data loss.**
This is what was done and why each step exists.

---

## 1. The Analogy (use this throughout)

When Helm installs something, it leaves **two kinds of claim** behind. They are
completely separate and neither one knows about the other.

### The Deed 📜

A Secret named `sh.helm.release.v1.<release>.v<N>`, sitting in the namespace.

This is the legal document in Helm's filing cabinet saying *"I own this."* It is
what lets Helm **act**:

| Command | What it does with the deed |
| --- | --- |
| `helm list` | reads it |
| `helm upgrade` | reads it, renovates the property |
| `helm rollback` | reads an older version of it, restores a previous layout |
| `helm uninstall` | reads it, **demolishes the property**, then shreds it |

**No deed → Helm has no idea the property exists.**

### The Nameplate 🪧

Three fields stamped onto the object itself:

```yaml
annotations:
meta.helm.sh/release-name: postgres-db
meta.helm.sh/release-namespace: student-api
labels:
app.kubernetes.io/managed-by: Helm
```

This is the plate screwed to the front door: *"maintained by Helm release
postgres-db."*

Helm **checks this before doing any work.** If the nameplate doesn't match a deed
it holds, Helm refuses to touch the property.

> [!important] The one sentence that explains the whole retirement order
> **The deed gives Helm the power to act. The nameplate is a check Helm performs
> on itself before acting.**

### Therefore: shred the deed first, unscrew the nameplate second

Picture someone accidentally running `helm upgrade` in the gap between the two
steps.

**Deed shredded first (correct):**
1. Helm checks the cabinet → no deed → *"I don't own this, must be a new build"*
2. Helm goes to build → finds a property already standing
3. Helm reads the nameplate → *"belongs to release postgres-db"*
4. Helm can't prove it holds that deed → **refuses. Changes nothing.**

The nameplate is a last line of defence during the gap.

**Nameplate removed first (wrong):**
1. Helm checks the cabinet → **deed is still there** → *"I own this"*
2. Helm goes to the property → no nameplate to contradict it
3. **Helm starts renovating**, fighting Argo CD for control

Same two actions. Opposite safety. The nameplate is worthless while the deed
exists, and valuable the moment it's gone.

> [!danger] Never use `helm uninstall`
> It doesn't shred the deed — **it demolishes the property and then shreds the
> deed.** Your StatefulSet, Service and ConfigMap are deleted.
>
> Helm 4 has `--keep-history` (keeps the deed, **still demolishes**). There is no
> `--keep-resources`. **There is no safe version of `helm uninstall` here.**
>
> `kubectl delete secret` does exactly one thing and you can see what it is.

---

## 2. What Makes These Harder Than a Config Object

| | `eso-config` (Part 1) | `student-api` | `postgres-db` |
| --- | --- | --- | --- |
| Objects | 1 | 4 | 4 |
| Runs pods | ❌ | ✅ Deployment | ✅ StatefulSet |
| Has data on disk | ❌ | ❌ | ✅ **1Gi PVC** |
| Immutable fields | none | `spec.selector` | `spec.selector`, `serviceName`, `volumeClaimTemplates` |
| Worst case if wrong | store stops refreshing | app restarts | **empty database serving traffic** |

Three new failure modes to defend against.

### 2.1 The release name is FROZEN — it names your objects

This is the biggest trap, and it is worse than it looks.

These charts use `{{ .Release.Name }}` for **object names**, not just labels:

```
helm/student-api/templates/deployment.yaml:4 name: {{ .Release.Name }}
helm/student-api/templates/service.yaml:4 name: {{ .Release.Name }}
helm/student-api/templates/configmap.yaml:4 name: {{ .Release.Name }}-config
helm/postgres-db/templates/statefulset.yaml:4 name: {{ .Release.Name }}
helm/postgres-db/templates/statefulset.yaml:10 serviceName: {{ .Release.Name }}
```

So a different release name doesn't *modify* your objects — it renders a
**completely different set of objects**.

Real dry-run with `releaseName: my-cool-api`:

```
configmap/my-cool-api-config serverside-applied (server dry run)
deployment.apps/my-cool-api serverside-applied (server dry run)
externalsecret/my-cool-api-db-url serverside-applied (server dry run)
The Service "my-cool-api" is invalid: spec.ports[0].nodePort: 30080 already allocated
```

**Three of four succeeded.** Argo CD would create a parallel stack with its own
pods, while your real objects keep running owned by nobody. Only the NodePort
collision caught it — pure luck.

> [!warning] An immutable-field error would be a loud, safe failure. This is
> **silent duplication**, which is much worse.

**On `postgres-db` this becomes a data incident:**

```
StatefulSet name = {{ .Release.Name }}
PVC name = <volumeClaimTemplate>-<statefulset>-<ordinal>
= data-postgres-db-0 ← your actual data
```

Rename the release to `pg-new` → StatefulSet `pg-new` → PVC **`data-pg-new-0`** →
a brand-new **empty** 1Gi volume. Postgres starts with no rows and serves traffic.
Your real data sits on the orphaned `data-postgres-db-0`, intact but disconnected.

**Nothing errors. Nothing alerts. The app just has no data.**

→ **Always pin `helm.releaseName` to the existing Helm release name. Never change it.**

### 2.2 App name and release name are different things — and only one is free

| | What it is | Free to change? |
| --- | --- | --- |
| Application `metadata.name` | Argo CD's own label for the app | ✅ **Yes** |
| `spec.source.helm.releaseName` | fills `{{ .Release.Name }}` in the chart | ❌ **Frozen forever** |

Pinning `releaseName` is precisely **what lets you** name the Application whatever
you like. Without the pin, `releaseName` silently defaults to the Application name
and the two get welded together.

Proof — the `postgres-db` chart deployed under an Application called `database`:

```
Application: database
releaseName: postgres-db

tracking-id: database:apps/StatefulSet:student-api/postgres-db
^^^^^^^^ app name ^^^^^^^^^^^ object name from release name
```

The Application name appears **only** in the tracking annotation. That's also why
renaming an app *later* is a real operation (see §7.2).

### 2.3 Workloads can restart — `generation` is your instrument

`metadata.generation` increments **only when `spec` changes.** For a Deployment or
StatefulSet, a `spec.template` change means a **rollout**.

So the test for "did the adoption disturb my workload?" is:

```bash
kubectl get statefulset postgres-db -n student-api -o jsonpath='{.metadata.generation}{"\n"}'
kubectl get pods -n student-api -l app.kubernetes.io/name=postgres-db \
-o custom-columns='NAME:.metadata.name,UID:.metadata.uid,START:.status.startTime,RESTARTS:.status.containerStatuses[0].restartCount'
```

**Same generation + same pod UID + same start time = the workload was never
touched.** Pod UID matters more than pod name: a StatefulSet replacement reuses
the name `postgres-db-0` but gets a **new UID**.

---

## 3. The Namespace Trap (this one bit me)

The go/no-go check before any migration is:

```bash
helm template <release> <chart> -n <ns> | kubectl diff -n <ns> -f -
echo $? # bash. fish: echo $status
```

**Both `-n` flags are required, and they do different things.**

I ran it without the second one and got garbage:

```
--- postgres-db --- exit=1 (diff found)
--- student-api --- exit=2 (ERROR)
The Service "student-api" is invalid: spec.ports[0].nodePort:
Invalid value: 30080: provided port is already allocated
```

Both were false alarms. Here's why:

```bash
kubectl config view --minify -o jsonpath='{..namespace}' # → argocd
helm template postgres-db helm/postgres-db -n student-api | grep -c "^ namespace:" # → 0
```

- `helm template -n student-api` sets `.Release.Namespace` **for the render**
- But these charts **never write `metadata.namespace`** into the output
- So `kubectl diff` used my *current context namespace* (`argocd`)
- It compared against objects that don't exist there → "everything is new"
- `student-api` errored because it tried to create a duplicate NodePort Service

Corrected:

```
--- postgres-db --- exit=0
--- student-api --- exit=0
```

> [!important] This is the same mechanism Argo CD relies on
> Because the charts don't stamp `metadata.namespace`, **Argo CD supplies it from
> `spec.destination.namespace`.** For `eso-config` that field did nothing (the
> object was cluster-scoped). Here it is load-bearing — get it wrong and objects
> land in the wrong namespace.

---

## 4. Runbook

Identical for both. Substitute the names from this table:

| | `student-api` | `postgres-db` |
| --- | --- | --- |
| Namespace | `student-api` | `student-api` |
| Chart path | `helm/student-api` | `helm/postgres-db` |
| `releaseName` | `student-api` | `postgres-db` |
| Objects | `configmap/student-api-config`<br>`service/student-api`<br>`deployment/student-api`<br>`externalsecret/student-api-db-url` | `configmap/postgres-db-config`<br>`service/postgres-db`<br>`statefulset/postgres-db`<br>`externalsecret/postgres-db` |

**Do `student-api` first.** It's stateless — if it goes wrong you restart an app.
`postgres-db` holds your data; do it once you've seen the pattern work.

---

### Step 1 — Baseline (read-only)

```bash
NS=student-api
mkdir -p /tmp/pg-migration

# THE GATE. Must be 0. Note BOTH -n flags.
helm template postgres-db helm/postgres-db -n $NS | kubectl diff -n $NS -f - >/dev/null 2>&1
echo "drift gate exit=$?"

# How many deeds? Helm keeps ONE PER REVISION.
kubectl get secret -n $NS -l owner=helm,name=postgres-db

# Photocopy the deed — this is the undo button
kubectl get secret -n $NS -l owner=helm,name=postgres-db -o yaml > /tmp/pg-migration/helm-backup.yaml

# Object specs. NOTE: ConfigMaps have .data, not .spec
kubectl get cm postgres-db-config -n $NS -o jsonpath='{.data}' | python3 -m json.tool \
> /tmp/pg-migration/before-configmap.json
for k in service/postgres-db statefulset/postgres-db externalsecret/postgres-db; do
kubectl get $k -n $NS -o jsonpath='{.spec}' | python3 -m json.tool \
> /tmp/pg-migration/before-$(basename $k).json
done

# Workload identity
kubectl get statefulset postgres-db -n $NS -o jsonpath='{.metadata.generation}{"\n"}' \
> /tmp/pg-migration/before-generation
kubectl get pods -n $NS -l app.kubernetes.io/name=postgres-db \
-o custom-columns='NAME:.metadata.name,UID:.metadata.uid,START:.status.startTime,RESTARTS:.status.containerStatuses[0].restartCount' \
> /tmp/pg-migration/before-pods
```

**Stateful extras — prove the data exists before you start:**

```bash
kubectl get pvc data-postgres-db-0 -n $NS -o jsonpath='{.spec.volumeName}{"\n"}' \
> /tmp/pg-migration/before-pv

kubectl exec -n $NS postgres-db-0 -- psql -U postgres -d students_db -c '\dt' \
> /tmp/pg-migration/before-tables.txt
kubectl exec -n $NS postgres-db-0 -- psql -U postgres -d students_db -t \
-c "select relname, n_live_tup from pg_stat_user_tables order by relname;" \
> /tmp/pg-migration/before-rows.txt
```

> [!tip] Insert a canary row
> If your tables are empty, the data proof is thin. Insert a known row first so
> "the data survived" means something concrete.

> [!danger] If the drift gate is not 0, STOP
> The chart in git no longer matches what Helm deployed. Migrating now means
> **adopting an unknown state.** Fix it with Helm, in Helm's world, first.

---

### Step 2 — Create the Application. Do not sync.

UI → **+ NEW APP** → **EDIT AS YAML** (the wizard has no Release Name field):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
name: database # free — call it whatever you like
namespace: argocd
spec:
project: default
source:
repoURL: https://github.com/madhav-wakhare/SRE-Bootcamp.git
targetRevision: argocd-migration
path: helm/postgres-db
helm:
releaseName: postgres-db # FROZEN — names the StatefulSet, which names the PVC
valueFiles:
- values.yaml
destination:
server: https://kubernetes.default.svc
namespace: student-api # load-bearing — the chart doesn't set it
syncPolicy:
syncOptions:
- ServerSideApply=true
```

**Deliberately absent:**

| Missing | Why |
| --- | --- |
| `automated:` | Manual sync. See the diff before it happens. |
| `resources-finalizer` | Without it, `kubectl delete app` stops reconciliation and **leaves everything running**. That is your rollback. |
| `helm.parameters` | `values.yaml` in git is the only source of values. |

> [!warning] Two UI wizard traps
> **1.** The HELM panel has no *Release Name* field — use **EDIT AS YAML**.
> **2.** The PARAMETERS fields look pre-filled but are **empty overrides showing
> current values as placeholders**. Type in one and Argo CD writes it to
> `spec.source.helm.parameters` — config that lives in the cluster, not in git,
> overriding `values.yaml`, invisible unless you read the Application spec.
> **Leave them all alone.**

**Verify the spec has nothing you didn't intend:**

```bash
kubectl get app database -n argocd -o jsonpath='{.spec}' | python3 -m json.tool
```

Check `releaseName` is right, and that `helm.parameters` and `syncPolicy.automated`
are both absent.

---

### Step 3 — Read the diff before syncing

```bash
kubectl get app database -n argocd
kubectl get app database -n argocd -o jsonpath='{.status.resources}' | python3 -m json.tool
```

UI → open the app → StatefulSet tile → **DIFF**.

**Expected: four tracking annotations and nothing else.**

> [!danger] Stop if you see any of these paths in the diff
> - `spec.volumeClaimTemplates` — **immutable**. Can never sync, and "fixing" it
> with `Replace=true` destroys the PVC.
> - `spec.selector`, `spec.serviceName` — **immutable**.
> - `spec.template` — not immutable, but it means a **pod restart**, i.e. a
> database bounce.

**The PVC check — do this before ever enabling prune:**

```bash
kubectl get app database -n argocd -o jsonpath='{.status.resources}' \
| python3 -m json.tool | grep -i persistentvolumeclaim
```

**Empty output is the answer you want.** The PVC was created by the StatefulSet
controller from `volumeClaimTemplates` — it is not in the chart, so not in git, so
Argo CD never tracks it, so **prune can never reach it.** Same for the Secret ESO
materialises from the ExternalSecret.

---

### Step 4 — Sync, then prove nothing moved

```bash
NS=student-api

# Claimed?
for k in configmap/postgres-db-config service/postgres-db statefulset/postgres-db externalsecret/postgres-db; do
echo -n "$k -> "
kubectl get $k -n $NS -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/tracking-id}{"\n"}'
done

# Spec untouched?
kubectl get statefulset postgres-db -n $NS -o jsonpath='{.spec}' | python3 -m json.tool \
> /tmp/pg-migration/after-sts.json
diff /tmp/pg-migration/before-statefulset.json /tmp/pg-migration/after-sts.json \
&& echo "STATEFULSET SPEC UNCHANGED"

# Workload untouched?
echo "generation before: $(cat /tmp/pg-migration/before-generation)"
echo "generation after : $(kubectl get statefulset postgres-db -n $NS -o jsonpath='{.metadata.generation}')"
cat /tmp/pg-migration/before-pods
kubectl get pods -n $NS -l app.kubernetes.io/name=postgres-db \
-o custom-columns='NAME:.metadata.name,UID:.metadata.uid,START:.status.startTime,RESTARTS:.status.containerStatuses[0].restartCount'

# Data untouched?
diff /tmp/pg-migration/before-pv <(kubectl get pvc data-postgres-db-0 -n $NS -o jsonpath='{.spec.volumeName}{"\n"}') \
&& echo "SAME UNDERLYING VOLUME"
kubectl exec -n $NS postgres-db-0 -- psql -U postgres -d students_db -c '\dt'
kubectl exec -n $NS postgres-db-0 -- psql -U postgres -d students_db -t \
-c "select relname, n_live_tup from pg_stat_user_tables order by relname;"
```

**What a clean adoption looks like — actual output from this migration:**

```
generation 1 → 1 spec never changed → no rollout
pod UID cfff90cf… → cfff90cf… same pod object, not a replacement
start time 2026-08-03T17:33:34Z unchanged → postgres never restarted
restarts 0
PV pvc-eff6a66c… → identical
tables alembic_version, students identical
rows 1, 0 identical
STATEFULSET SPEC UNCHANGED
```

---

### Step 5 — Retire Helm (deed first, nameplate second)

```bash
NS=student-api

# Confirm Argo CD is genuinely in charge BEFORE removing your fallback
kubectl get app database -n argocd # Synced / Healthy

# How many deeds again? (one per revision)
kubectl get secret -n $NS -l owner=helm,name=postgres-db

# 📜 Shred the deed
kubectl delete secret -n $NS -l owner=helm,name=postgres-db
helm list -n $NS # postgres-db is gone

# 🪧 Unscrew the nameplate — on ALL FOUR objects
for k in configmap/postgres-db-config service/postgres-db statefulset/postgres-db externalsecret/postgres-db; do
kubectl annotate $k -n $NS meta.helm.sh/release-name- meta.helm.sh/release-namespace-
kubectl label $k -n $NS app.kubernetes.io/managed-by-
done
```

The trailing `-` is `kubectl` syntax: `key=value` sets, `key-` removes.

> [!warning] Never remove `app.kubernetes.io/name` or `app.kubernetes.io/instance`
> Those come from the **chart**, not from Helm. Argo CD renders the same chart, so
> it puts them straight back within a minute.
>
> The ownership table tells you which is which:
> ```
> app.kubernetes.io/managed-by helm ← chart doesn't have it → safe to remove
> app.kubernetes.io/instance argocd-controller, helm ← chart has it → LEAVE ALONE
> ```
> **Helm-only = remove. Shared = leave.**

> [!danger] On workloads, check the selector before removing any label
> `spec.selector` on a Deployment/StatefulSet is **immutable**. Remove a label that
> the selector matches on and you orphan every pod — with no way to fix it forward.
>
> Here the selector is `{instance: postgres-db, name: postgres-db}`. `managed-by`
> isn't in it, so removing it is safe. **Verify this yourself every time:**
> ```bash
> kubectl get statefulset postgres-db -n $NS -o jsonpath='{.spec.selector.matchLabels}{"\n"}'
> ```

**Verify after:**

```bash
kubectl get app database -n argocd # still Synced
kubectl get statefulset postgres-db -n $NS -o jsonpath='generation={.metadata.generation}{"\n"}' # still 1
kubectl get pods -n $NS -l app.kubernetes.io/name=postgres-db \
-o custom-columns='NAME:.metadata.name,UID:.metadata.uid,RESTARTS:.status.containerStatuses[0].restartCount'
kubectl exec -n $NS postgres-db-0 -- psql -U postgres -d students_db -c '\dt'
```

> [!note] Removing `managed-by` is not drift
> That label was never in git, so Argo CD has no opinion about it. If the app went
> `OutOfSync` here, it would mean you deleted something the chart **does** render —
> a real signal worth stopping for.

---

### Undo, if you need it

```bash
NS=student-api
kubectl apply -f /tmp/pg-migration/helm-backup.yaml
for k in configmap/postgres-db-config service/postgres-db statefulset/postgres-db externalsecret/postgres-db; do
kubectl annotate $k -n $NS \
meta.helm.sh/release-name=postgres-db meta.helm.sh/release-namespace=$NS
kubectl label $k -n $NS app.kubernetes.io/managed-by=Helm
done
```

Deed back in the cabinet, nameplate back on the door. `helm list` shows the release
and `helm rollback` works again. **Test this path before you need it.**

---

## 5. Never Set These On A Stateful App

| Setting | Why not |
| --- | --- |
| `Replace=true` | Does `kubectl replace` — for a StatefulSet with a bound PVC that means **delete and recreate**. Data loss. |
| `prune: true` (before checking) | Only safe once you've confirmed Argo CD doesn't track the PVC. Run the §4 Step 3 check first. |
| `resources-finalizer` (during migration) | Makes `kubectl delete app` cascade into deleting the StatefulSet. That's the natural panic response — don't arm it. |
| Changing `helm.releaseName` | Renames the StatefulSet, which renames the PVC, which gives you an empty database. Silently. |
| Changing `destination.namespace` | Changes the tracking ID on every object → Argo CD stops recognising them as its own. |

---

## 6. Result

```
helm list -A
argocd argocd ← Argo CD itself, still Helm-managed
external-secrets eso-ns ← intentionally out of scope
vault vault ← not migrated yet

kubectl get app -n argocd
eso-config Synced Healthy
student-api Synced Healthy
database Synced Healthy
```

Three releases migrated. **Zero downtime. Zero pod restarts. Zero data loss.**

---

## 7. Loose Ends

### 7.1 The Applications only exist in the cluster

All three were created in the UI, so **nothing in git describes them.** The thing
enforcing GitOps isn't itself under GitOps — lose this cluster and you rebuild them
from memory.

Fix:

```bash
for a in eso-config student-api database; do
kubectl get app $a -n argocd -o yaml > gitops/apps/$a.yaml
done
```

Then strip `status`, `metadata.uid`, `resourceVersion`, `generation`,
`creationTimestamp` and `managedFields` from each, commit, and `kubectl apply -f`
becomes the workflow.

### 7.2 Renaming an Application later

Kubernetes object names are immutable, so there is no rename — you delete and
recreate. The tracking annotation on every resource carries the old app name.

```bash
# 1. Confirm no finalizer — without it, deleting the app leaves resources running
kubectl get app database -n argocd -o jsonpath='{.metadata.finalizers}{"\n"}'
# If not empty:
# kubectl patch app database -n argocd --type merge -p '{"metadata":{"finalizers":null}}'

# 2. Delete the old Application. Resources keep running, unmanaged.
kubectl delete app database -n argocd

# 3. Create the new one — new metadata.name, SAME releaseName, same source. Sync.
```

That's the same adoption you already did, just Argo→Argo instead of Helm→Argo.

> [!warning] Don't run both at once
> If old and new Applications both exist, both claim the same resources and each
> sync flips the tracking-id back and forth. **Delete first, then create.**

### 7.3 Nothing is on auto-sync yet

Every app is still manual — drift is reported but not corrected. Enable in this
order, **one at a time, verifying between each**:

```
1. auto-sync + selfHeal (prune: false)
2. prune: true (only after the PVC check)
3. resources-finalizer (last, and maybe never on the database)
```

**Why prune goes late:** prune deletes what Argo CD tracks but git no longer
renders. The danger isn't prune — it's an **empty render**. A bad values edit or a
broken branch makes the chart produce nothing, and *"git produces nothing"* looks
exactly like *"delete everything."* It is the only irreversible switch in an
Application.

Test self-heal once it's on:

```bash
kubectl patch cm postgres-db-config -n student-api --type merge \
-p '{"data":{"POSTGRES_DB":"wrong"}}'
sleep 15
kubectl get cm postgres-db-config -n student-api -o jsonpath='{.data.POSTGRES_DB}{"\n"}'
```

Heals in ~2s → self-heal's watch caught it. Closer to a minute → the git poll did,
which means self-heal isn't actually on.

> [!tip] Pick the ConfigMap, not the StatefulSet, for the drift test
> Drifting the StatefulSet would trigger a rollout when it heals — a database
> restart to prove a point.

---

## 8. Checklist

### Before
- [ ] `helm template <rel> <chart> -n <ns> | kubectl diff -n <ns> -f -` → **exit 0** (both `-n` flags!)
- [ ] Count the deeds: `kubectl get secret -n <ns> -l owner=helm,name=<rel>`
- [ ] Deed backed up
- [ ] Grep the chart for `randAlphaNum`, `lookup`, `helm.sh/hook`
- [ ] Note every immutable field: `spec.selector`, `serviceName`, `volumeClaimTemplates`
- [ ] Record `generation`, pod **UID**, pod start time, restart count
- [ ] **Stateful:** record PV name, table list, row counts. Insert a canary row.

### Application
- [ ] `helm.releaseName` = the **existing Helm release name**, pinned
- [ ] `destination.namespace` correct (load-bearing for namespaced resources)
- [ ] `ServerSideApply=true`
- [ ] No `automated:`, no finalizer, no `helm.parameters`
- [ ] Diff reviewed — **only tracking annotations**
- [ ] Nothing under `volumeClaimTemplates`, `selector`, `serviceName`, `template`
- [ ] PVC **not** in `status.resources`

### After sync
- [ ] All objects carry `argocd.argoproj.io/tracking-id`
- [ ] Specs unchanged (diff the files)
- [ ] `generation` unchanged
- [ ] Pod **UID** and start time unchanged, restarts still 0
- [ ] **Stateful:** same PV, same tables, same row counts, canary row present
- [ ] App healthy end-to-end (`make helm-verify`)

### Retire Helm
- [ ] Argo CD `Synced / Healthy` **first**
- [ ] 📜 Deed deleted (**never `helm uninstall`**)
- [ ] `helm list` no longer shows the release
- [ ] Selector checked before removing any label
- [ ] 🪧 Nameplate removed from **every** object
- [ ] `app.kubernetes.io/name` and `instance` **left alone**
- [ ] App still `Synced`, `generation` still unchanged
- [ ] Undo path tested at least once

---

## 9. The Five Rules

1. **Make git match reality first.** A zero-diff baseline turns adoption into a
metadata change. That's what buys zero downtime — not careful timing.

2. **The release name is frozen; the app name is free.** The release name is baked
into your object names, and on a StatefulSet into your PVC name.

3. **`generation` is the truth about whether you disturbed a workload.** Same
generation + same pod UID = nothing happened.

4. **Deed before nameplate.** Take away the power to act before you take away the
check that catches mistakes.

5. **One change per step.** Adopt → verify → retire Helm → automate → prune. When
something breaks you know exactly which change did it.

> [!quote]
> A good migration is boring. If it's exciting, something is wrong.