# CNPG Setup

CloudNativePG is a Kubernetes operator for PostgreSQL. It is designed for ease of use, automation, and reliability when deploying PostgreSQL clusters in Kubernetes.

## 1. Install CNPG

The easiest way to install CNPG is using `kubectl apply --server-side`, We will install operator in namespace `cnpg-system`.

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.29/releases/cnpg-1.29.0.yaml
```

**NOTE: This is the latest release at this time. You can check the latest release at [https://github.com/cloudnative-pg/cloudnative-pg/releases](https://github.com/cloudnative-pg/cloudnative-pg/releases)**

## 2. Verify CNPG is installed

Wait for a few minute to check pod of the operator is running.

```bash
kubectl get pods -n cnpg-system
```

We should see the pod name `cnpg-controller-manager-...` is running.

## 3. Create a PostgreSQL Cluster

When the operator is ready, we can create a PostgreSQL cluster. We will create a cluster.
It will use 'local-path' as a storage class.

Create file `cluster-example.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cluster-example
  # You can change namespace as you want
  namespace: database
spec:
  instances: 3 # Number of Pods (1 Primary, 2 Replicas). You can change as you want.
  storage:
    size: 1Gi # Storage size of each Pod. You can change as you want.
    # k3s will use 'local-path' as a default storage class automatically.
```

Apply the config:

```bash
kubectl apply -f cluster-example.yaml
```

## 4. Check database status

You can see cluster creating status by following commands:

```bash
kubectl get cluster -n <namespace>
```

***NOTE: Change to namespace that you have configured or leave it for default namespace***

And check PostgreSQL Pod status:

```bash
kubectl get pods -l cnpg.io/cluster=<cluster-name> -n <namespace>
```

You should see the PostgreSQL Pod is running (X/X READY).

## How to connect to the database

CNPG will create secret that store password for us automatically. So you can check password by following command:

```bash
kubectl get secret -n <namespace> <cluster-name>-app -o jsonpath='{.data.password}' | base64 --decode
```

Forward port for temporary connection.

```bash
kubectl port-forward svc/<cluster-name>-rw -n <namespace> 5432:5432
```

After that, you can use any tool like `DBeaver`, `pgAdmin` or `psql` to connect to the database.

- HOST: `localhost:5432`
- User: `app`
- Password: (copy from the command above)
- Database: `app`
