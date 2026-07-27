Right now, look at how a new build actually reaches your cluster:

1. You push code → CI ([.github/workflows/ci.yml] builds `wakharemadhav/sre-student-api:<commit-sha>` and pushes it to Docker Hub.
2. ...and then nothing. Nobody's laptop knows a new image exists. `helm/student-api/values.yaml`'s `image.tag` still points at whatever was there before.
3. To actually deploy that new image, **you** — a human, sitting at a terminal with `kubectl` and `helm` configured — have to notice the new tag, edit `values.yaml` (or pass `--set image.tag=...`), and run `helm upgrade --install student-api ...` yourself.

That's the exact gap: **CI can build and push, but nothing closes the loop back into the cluster.** ArgoCD is what closes it.

## What GitOps actually changes about this

The core idea: instead of "a human runs `helm upgrade` from their laptop," you get a controller **running inside the cluster** whose entire job is to continuously ask one question: _"does what's live in the cluster match what's declared in Git?"_ If not, it reconciles — pulls the Helm chart + values from your repo, renders it, and applies it. Nobody ever runs `helm upgrade` by hand again; the only "deploy trigger" that exists is **a Git commit**.

This is why the milestone insists on two things that might look like bureaucracy but are actually the whole point:

- **"Helm charts and values.yaml are the source of truth"** — meaning the only way to change what's running is to change a file in this repo. Not `kubectl edit`, not `helm upgrade` typed by a person.
- **"ArgoCD components created declaratively, not manually"** — this is the same principle applied to ArgoCD _itself_. If you `kubectl apply` an ArgoCD `Application` object by hand once and never commit it anywhere, then Git _isn't actually_ the full source of truth — there's a second, invisible source of truth sitting in whoever's shell history ran that command. Every ArgoCD `Application` has to be a YAML file checked into this repo too, so the whole system — app config included — is reconstructable from Git alone.

## Why this connects directly to your existing CI

You already have a self-hosted-runner CI pipeline that builds and pushes an image tagged with the commit SHA. The new job this milestone asks for (bump `values.yaml`, commit) is the **only missing link** between "CI produced a new image" and "ArgoCD notices a Git change and rolls it out." Once that job exists:

```
push code → CI builds+pushes image → CI job bumps values.yaml → commits →
ArgoCD polls/watches the repo → sees the diff → auto-syncs → new pod running
```

No step in that chain involves a person touching `kubectl` or `helm` directly, ever again.

ArgoCD is installed as k8s controller with k8s environment. The ArgoCD API Server is a gRPC REST API Server which exposes the API.
We can configure a webhook in github to notify argocd about events in github.


Once the repositories are set up, an operator like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) is deployed inside the Kubernetes cluster. ArgoCD continuously monitors the manifests repository for changes and reconciles the actual state of the cluster with the desired state by automatically deploying updates and managing resource lifecycle events.

Installation Options:
![[Pasted image 20260721120510.png]]


```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace if not exists
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

# Install/Upgrade ArgoCD
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --values argocd/helm-values.yaml \
  --wait

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

echo "ArgoCD installed successfully!
```

## The actual pieces inside ArgoCD

When you install ArgoCD (via its official manifests or Helm chart), you get several Deployments/StatefulSets working together — not one monolithic "ArgoCD" pod:

|Component|What it actually does|
|---|---|
|**`argocd-repo-server`**|Clones this Git repo, and _renders_ the manifests — for a chart, that means running the equivalent of `helm template ./helm/student-api -f values.yaml`. It never talks to the Kubernetes API to apply anything — its only job is "turn Git contents into plain YAML."|
|**`argocd-application-controller`**|The actual brain. It reads `Application` objects (see below), asks `repo-server` for the rendered YAML, compares that against what's _actually_ running in the cluster, and — if auto-sync is on — applies the diff via the Kubernetes API. This is the reconciliation loop that makes "commit = deploy" true.|
|**`argocd-server`**|The API server + web UI + what the `argocd` CLI talks to. This is how _you_ would look at sync status, but it's not required for the reconciliation itself to work.|
|**`argocd-redis`**|Internal cache the other components share (rendered-manifest cache, etc.). Required even in a minimal install.|
|**`argocd-dex-server`**|SSO/OIDC login (GitHub, Google...). Not needed for a bootcamp setup — we'd disable it.|
|**`argocd-applicationset-controller`** / **`argocd-notifications-controller`**|Optional extras (bulk-generate Applications from a template; send Slack/webhook notifications on sync). Not required, but `applicationset-controller` is relevant to a design choice I'll flag below.|

All of these get a `nodeSelector: {type: dependent_services}` and live in the `argocd` namespace — same placement pattern already established for `vault`. That part's simple and matches what's already there (empty `argocd` namespace, created but nothing deployed).

## The two-to-three things we actually write as files in Git

This is the part the milestone's "declarative, not manual" requirement is really about — not the ArgoCD installation itself (that's just another Helm chart, same pattern as `external-secrets`), but these:

1. **`Application` objects — one per thing we want ArgoCD to manage.** Each one is a small YAML file saying: "watch _this_ path in _this_ repo, render it with Helm, apply it to _this_ namespace, and here's the sync policy (auto-sync on push? self-heal if someone manually `kubectl edit`s something? prune resources that got removed from Git?)." We'd have one per existing chart: `external-secrets`, `vault`, `eso-config`, `postgres-db`, `student-api`.
    
    _(Design question for later: 5 separate `Application` files, or one `ApplicationSet` that generates all 5 from a list? Not deciding now — just flagging that `applicationset-controller` from the table above is what that second option would need.)_
    
2. **Repository access.** ArgoCD needs to know _how to reach this Git repo_ at all — if it's public, that's nearly nothing (just a URL). If it's private, ArgoCD needs a credential (a GitHub token or deploy key) stored as a Kubernetes `Secret` with a specific label ArgoCD looks for. This is where "declarative" gets genuinely interesting: you can't just commit a plaintext GitHub token into this repo. We have two realistic options here — either a manually-created-once Secret (a small, deliberate exception to "no manual creation," common even in strict GitOps setups since _some_ root credential has to originate somewhere), or — since you already have Vault + ESO built for exactly this kind of problem — syncing this credential in from Vault the same way `postgres-db`/`student-api` do. Worth deciding once we know the repo's visibility.
    
3. _(Possibly)_ **`AppProject`** — a way to restrict what an `Application` is allowed to do (which repos, which destination namespaces/clusters, which resource kinds). Optional for a single-repo, single-cluster bootcamp setup — the built-in `default` project covers it — but worth knowing it exists.
    

That's the shape of "what." Two things I'd like your input on before we move to "how":

1. **Is this repo public or private on GitHub?** — settles the repo-credentials question above.
2. **Do you want 5 separate `Application` files, or 1 `ApplicationSet`?** — I lean toward starting with 5 explicit files since it's easier to reason about individually while learning, and revisiting `ApplicationSet` later once the pattern is familiar — but this is genuinely your call, not a right/wrong.


### Terminologies :
![[Pasted image 20260727200749.png]]


ArgoCD also exposes different sets of Prometheus Metrics, which can be visualized through Grafana & it also have out-of box notification service with various triggers. 