# GitOps Setup with ArgoCD

This guide provides step-by-step instructions to set up GitOps using ArgoCD. GitOps is a powerful approach to managing and deploying applications using Git as the single source of truth.

We will install by separate namespaces for ArgoCD and the applications, and we will use a Git repository to store our application manifests.

## 1. Install ArgoCD

Create `argocd` namespace and install ArgoCD in it:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Waiting a few minutes for all Pods to be up and running:

```bash
kubectl get pods -n argocd
```

## 2. Configure ArgoCD to insecure mode

By default, ArgoCD API server uses TLS with self-signed certificates, which can cause issues when accessing the UI. To simplify access, we will configure ArgoCD to run in insecure mode.

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.insecure":"true"}}'
```

Rollout the deployment to apply the changes:

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

## 3. Access ArgoCD UI

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Forwards the ArgoCD API server to your local machine:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Go to `http://localhost:8080` in your web browser and log in with username `admin` and the password you retrieved in the previous step.

## 4. Expose ArgoCD API Server with Cloudflare Tunnel

Create `argocd-ingress.yaml` with the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
    - host: argocd.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```

```bash
kubectl apply -f argocd-ingress.yaml
```

Now you can access ArgoCD UI at `https://argocd.yourdomain.com` using the same credentials as before.
