# Vault Setup

We will use HashiCorp Vault as a secret manager for ArgoCD and other applications in the cluster.

Before we start, we need to install `vault` command line tool. [HashiCorp Vault Installation](https://developer.hashicorp.com/vault/install)

## Vault Cluster Installation

We will install vault standalone using helm chart from [Helm Charts](https://github.com/hashicorp/vault-helm).

### 1. Add HashiCorp Helm Repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com/
helm repo update
```

### 2. Install Vault

Create `values.yaml` with the following configuration:

```yaml
server:
  # Configure Storage (K3s normally has a local-path storage class)
  dataStorage:
    enabled: true
    size: 2Gi 

  # Limit Resource to prevent it from affecting other apps in the VM
  resources:
    requests:
      memory: 128Mi
      cpu: 50m
    limits:
      memory: 256Mi
      cpu: 250m

ui:
  # Enable Web UI for ease of management
  enabled: true
  serviceType: "ClusterIP"
```

Run helm install command:

```bash
kubectl create namespace vault
helm install vault hashicorp/vault -n vault -f values.yaml
```

Check the vault pod status:

```bash
kubectl get pods -n vault
```

You will see pod `vault-0` in `Running` state but `Not Ready (0/1)`. So we need to unseal the vault.

### 3. Unseal Vault (Important)

Vault has been design to sealed every time it restarts. So we need to unseal it every time it restarts.

We need to unseal the vault using the unseal key.

#### 3.1 Initialize Vault

Run the following command to initialize the vault:

```bash
kubectl exec -it vault-0 -n vault -- vault operator init
```

***IMPORTANT***: You must save the `Unseal Key` and `Initial Root Token` somewhere safe. Because this will not be shown again.

#### 3.2 Unseal Vault

To unseal the vault, you need to provide the unseal key. Run the following command to unseal the vault for 3 times. You can use any 1 of 5 Unseal Keys:

```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal
```

After 3 unseals, vault will be unsealed. Run the following command to verify the status. YOu will see pod `vault-0` change the status to `Ready (1/1)`:

```bash
kubectl exec -it vault-0 -n vault -- vault status
```

### 4. Access to Vault Web UI

You can port-forward the vault-ui service to access the web ui locally.

```bash
kubectl port-forward svc/vault-ui -n vault 8200:8200
```

Go to `http://localhost:8200` and login using `Initial Root Token` you saved earlier to access vault web ui.

## Configure GitHub Token Secret for ArgoCD

To store github token to vault, there are many options. You can use the vault web ui or use vault cli command. So we will continue with `vault` command line.

### Part 1

### 1. Login to Vault and enable secret engine

After we install `vault` command line. We need to login to vault first. Using `port-forward` created earlier.

```bash
export VAULT_ADDR='http://localhost:8200'
vault login <YOUR_ROOT_TOKEN>
```

Enable KV-V2 engine to path name `secret`

```bash
vault secrets enable -path=secrets
```

### 2. Save GitHub token to Vault

Run the following command to save the github token to vault.

```bash
vault kv put secret/argocd/gitops-repo token="<YOUR_GITHUB_TOKEN>"
```

***NOTE:*** In the command if you want to save other secret like `username` follow this format:

```bash
vault kv put secret/argocd/gitops-repo token="xxxx" username="myuser"
```

### 3. Check for secrets in Vault

Run the following command to check the secrets in vault:

```bash
vault kv get secret/argocd/gitops-repo
```

### 4. Create policy for read secret

Create file `argocd-policy.hcl`.

```hcl
path "secret/data/argocd/*" {
  capabilities = ["read"]
}
```

And then apply the policy:

```bash
vault policy write argocd-policy - < argocd-policy.hcl
```

### 5. Enable Kubernetes Auth Method

Run the following command to enable Kubernetes auth method:

```bash
valt auth enable kubernetes

# Configure Vault to communicate with the Kubernetes cluster, Run this command in the machine where you run the kubectl command.
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc.cluster.local:443"
```

### 6. Create Role with Policy to Service Account

We will let `Service Account` from `argocd` namespace to have this policy:

```hcl
vault write auth/kubernetes/role/argocd-role \
    bound_service_account_names=argocd-server \
    bound_service_account_namespaces=argocd \
    policies=argocd-policy \
    ttl=24
```

### Part 2

We will use `ESO` as a bridge to sync information from Vault to K8s secrets.

#### 1. First, install `ESO` with Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

#### 2. Create SecretStore

Create file `vault-store.yaml` with the following configuration:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: argocd
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "argocd-role" # Role name that we create in Vault
          serviceAccountRef:
            name: "argocd-server" # Service Account name of ArgoCD
```

And then apply it:

```bash
kubectl apply -f vault-store.yaml
```

#### 3. Create ExternalSecret (make them sync data from Vault to K8s secret)

Create file `github-token-sync.yaml` with the following configuration, This ESO will sync `token` from Vault to `github-repo-creds` secret in `argocd` namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: github-token-sync
  namespace: argocd
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: github-repo-creds # Secret name that will show in K8s
    template:
      metadata:
        labels:
          argocd.argoproj.io/secret-type: repository # Label that let ArgoCD to recognize this secret
      data:
        url: "https://github.com/your-user/your-gitops-repo.git"
        password: "{{ .token }}" # Extract value from key 'token' in Vault
        username: "git" # Enter any username for GitHub Token
  data:
  - secretKey: token
    remoteRef:
      key: argocd/github-token
      property: token
```

Apply this yaml to create ESO.

```bash
kubectl apply -f github-token-sync.yaml
```

Check for secret `github-repo-creds` in `argocd` namespace:

```bash
kubectl get secret github-repo-creds -n argocd
```

### Part 3 Add Repo to ArgoCD

You can add repository to ArgoCD with these method:

#### 1. Using UI

1. Login to ArgoCD UI > Settings > Repositories
2. Connect Repo
3. Choose menu `Via HTTPS`
4. Fill URL: `"https://github.com/your-user/your-gitops-repo.git"`
5. Filed username and password, you can fill any string. So ArgoCD will use secret from ESO automatically.

#### 2. Using Declarative (Recommended)

Due to we fill label `argocd.argoproj.io/secret-type: repository` in ESO Secret, ArgoCD will automatically recognize this secret. So we can use declarative method to add repo to ArgoCD.

You can check on menu `Settings -> Repositories` in ArgoCD UI
