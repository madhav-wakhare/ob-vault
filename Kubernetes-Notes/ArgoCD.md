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