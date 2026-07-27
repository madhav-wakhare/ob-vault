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

echo "ArgoCD installed successfully!"