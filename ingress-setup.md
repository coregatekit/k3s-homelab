# Setup Ingress with Cloudflare Tunnel

## Overview

We will use cloudflare tunnel to expose our Kubernetes services to the internet. This allows us to securely route traffic through Cloudflare's network without needing to manage public IPs or load balancers.

```text
Internet
     ↓
Cloudflare Network
     ↓
Cloudflare Tunnel
     ↓
Traefik Ingress Controller
     ↓
Kubernetes Services
```

## Installation Steps

We will setup Cloudflare first, then install Traefik Ingress Controller in Kubernetes and configure it to work with Cloudflare Tunnel.

### Step 1: Create Cloudflare Tunnel

Create tunnel and token

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) and log in to your account.
2. Navigate to `Networks` > `Tunnels` > `Create a Tunnel`.
3. Choose `Cloudflared`  > Set tunnel name > Click `Save Tunnel`.
4. Navigate to next page you will see command to setup. Look for `Tunnel Token` (A long string `--token`) and copy it.
5. In your tunnel overview page, Go to `Routes` tab > Click `Add route` > Choose `Published application`
6. Subdomain: `*` > choose your domain > Service URL: `http://traefik.kube-system.svc.cluster.local:80` > click `Add route`

### Step 2: Install cloudflared on K3S node

Create secret for cloudflared tunnel token

```bash
kubectl create secret generic cloudflared-token \
  --from-literal=token='<CLOUDFLARED_TUNNEL_TOKEN>' \
```

Create deployment `cloudflared.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args: ["tunnel", "--no-autoupdate", "run", "--token", "$(TUNNEL_TOKEN)"]
        env:
        - name: TUNNEL_TOKEN
          valueFrom:
            secretKeyRef:
              name: cloudflared-token
              key: token
```

And then apply the deployment

```bash
kubectl apply -f cloudflared-deployment.yaml
```

### Step 3: Test with the first service

Try to start nginx deployment and expose it with a service

```bash
kubectl create deployment web-test --image=nginx
kubectl expose deployment web-test --port=80
```

Create ingress resource `web-test-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-test-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: test.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-test
            port:
              number: 80
```

Apply the ingress resource

```bash
kubectl apply -f web-test-ingress.yaml
```

Now you should be able to access `http://test.yourdomain.com` and see the nginx welcome page. This confirms that the Cloudflare Tunnel and Traefik Ingress Controller are working correctly together.
