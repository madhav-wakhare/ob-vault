

Here are all the exact commands executed inside the Vault pod (`vault-vault-0` or `vault-0` in namespace `vault-ns`) to initialize, configure, and seed HashiCorp Vault for External Secrets Operator (ESO):

helm install ev ./eso-vault/ --namespace vault --create-namespace
helm install app ./student-app/ --namespace student-app --create-namespace

---

### 1. Initialization & Unsealing
```bash
# 1. Initialize Vault and generate 1 root key share
vault operator init -key-shares=1 -key-threshold=1 -format=json

# 2. Unseal Vault using the generated unseal key
vault operator unseal <UNSEAL_KEY>
```

---

### 2. Enable & Configure Kubernetes Authentication Engine
```bash
# 1. Enable Kubernetes Auth Method
vault auth enable kubernetes

# 2. Configure Vault to talk to the local Kubernetes API server for TokenReview verification
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer="https://kubernetes.default.svc.cluster.local"
```

---

### 3. Create Access Control Policy (`eso-policy`)
```bash
# Define a policy that grants read permissions to secret paths
echo 'path "secret/data/*" { capabilities = ["read"] }' | vault policy write eso-policy -
```

---

### 4. Create Kubernetes Auth Role (`eso-role`)
```bash
# Bind the 'eso-policy' to the 'external-secrets' ServiceAccount in the 'external-secrets-ns' namespace
vault write auth/kubernetes/role/eso-role \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets-ns \
  policies=eso-policy \
  ttl=1h
```

---

### 5. Enable KV-v2 Engine & Seed Credentials
```bash
# 1. Enable Key-Value v2 Secrets Engine at path 'secret/'
vault secrets enable -path=secret kv-v2

# 2. Seed static secrets path
vault kv put secret/one2n/dev/app-config \
  db_password="changeme" \
  db_url="postgresql://postgres:changeme@post# Create the secret engine
kubectl exec -n vault vault-statefulset-0 -- vault secrets enable -version=2 -path=secret kv

# Create the policy for ESO
kubectl exec -n vault vault-statefulset-0 -- sh -c 'cat <<EOF > /tmp/eso-policy.hcl
path "secret/data/one2n/dev/app-config" {
  capabilities = ["read"]
}
EOF
vault policy write eso-policy /tmp/eso-policy.hcl'

# Add the actual secrets (Replace with your actual DB URL, password, and dockerhub config)
kubectl exec -n vault vault-statefulset-0 -- vault kv put secret/one2n/dev/app-config \
    db_url="postgresql://postgres:password@postgres-db-svc.student-app.svc.cluster.local:5432/students_db" \
    db_password="password" \
    dockerconfigjson='{"auths":{"https://index.docker.io/v1/":{"auth":"base64_encoded_auth"}}}'
