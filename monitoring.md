# Monitoring with LGTM Stack

## Installation Steps

### 1. Prepare environment and Helm repository

We will split monitoring to a separate namespace `lgtm` to keep it organized. You can choose any name you like.

Create namespace for monitoring stack:

```bash
kubectl create namespace lgtm
```

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Install Loki (Logs) in Single Binary mode

Loki in this mode will run all components (ingester, querier, distributor, etc.) in a single pod.

It will take 100-200MB of memory and 0.5-1 CPU, which is suitable for a home lab environment.

Create `loki-values.yaml` with the following content:

```yaml
deploymentMode: SingleBinary
loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
    path_prefix: /var/loki
  rulerConfig:
    storage:
      type: local
      local:
        directory: /var/loki/rules
    rule_path: /var/loki/rules-tmp
  storage:
    type: filesystem
    filesystem:
      chunks_directory: /var/loki/chunks
      rules_directory: /var/loki/rules
  schemaConfig:
    configs:
      - from: 2026-04-30
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h

singleBinary:
  replicas: 1

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0

lokiCanary:
  enabled: false

test:
  enabled: false

```

Run the following command to install Loki:

```bash
helm install loki grafana/loki -n lgtm -f loki-values.yaml
```

### 3. Install Tempo (Traces)

Create `tempo-values.yaml` with the following content:

```yaml
traces:
  otlp:
    grpc:
      enabled: true
    http:
      enabled: true
```

Run the following command to install Tempo:

```bash
helm install tempo grafana/tempo -n lgtm -f tempo-values.yaml
```

### 4. Install Prometheus (use Metrics instead Mimir)

We will install server only. Other components in not needed for a home lab environment.

Create file `prom-values.yaml` with the following content:

```yaml
alertmanager:
  enabled: false
prometheus-pushgateway:
  enabled: false
prometheus-node-exporter:
  enabled: false
server:
  retention: 5d # Retain metrics for 5 days, adjust as needed
```

Run the following command to install Prometheus:

```bash
helm install prometheus prometheus-community/prometheus -n lgtm -f prom-values.yaml
```

### 5. Install Grafana and configure data sources

We will configure Grafana to use Loki, Tempo and Prometheus as data sources.

Create `grafana-values.yaml` with the following content:

```yaml
ingress:
  enabled: true
  ingressClassName: traefik
  hosts:
    - grafana.yourdomain.com # Change to your domain
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server.lgtm.svc.cluster.local
        isDefault: true
      - name: Loki
        type: loki
        url: http://loki.lgtm.svc.cluster.local:3100
      - name: Tempo
        type: tempo
        url: http://tempo.lgtm.svc.cluster.local:3100
```

Run the following command to get the Grafana admin password:

```bash
kubectl get secret --namespace lgtm grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Go to `http://grafana.yourdomain.com` and log in with username `admin` and the password you just retrieved.

### 6. Install Alloy

We will install agent for collecting node metrics, logs and traces from pods.

Create `alloy-values.yaml` with the following content:

```yaml
# alloy-values.yaml
alloy:
  configMap:
    content: |-
      // 1. Define the destination: Send data to our Loki
      loki.write "local" {
        endpoint {
          url = "http://loki.lgtm.svc.cluster.local:3100/loki/api/v1/push"
        }
      }

      // 2. Discover: Scan all Pods running in k3s
      discovery.kubernetes "pods" {
        role = "pod"
      }

      // 3. Organize: Extract Namespace, Pod, and Container names as Labels (for easier search in Grafana)
      discovery.relabel "pod_logs" {
        targets = discovery.kubernetes.pods.targets
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
      }

      // 4. Act: Collect logs from the organized targets and send them to Loki
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pod_logs.output
        forward_to = [loki.write.local.receiver]
      }

      // 5. (Optional) Collect node metrics using Alloy's built-in Prometheus exporter for Unix system metrics
      prometheus.exporter.unix "node_stats" {
      }

      // 6. (Optional) Specify the destination to send Metrics (Prometheus Server)
      prometheus.scrape "node_metrics" {
        targets    = prometheus.exporter.unix.node_stats.targets
        forward_to = [prometheus.remote_write.local.receiver]
      }

      prometheus.remote_write "local" {
        endpoint {
          url = "http://prometheus-server.lgtm.svc.cluster.local/api/v1/write"
        }
      }
```

Run the following command to install Alloy:

```bash
helm install alloy grafana/alloy -n lgtm -f alloy-values.yaml
```

Check the results in Grafana. Waiting for a few minutes after installing Alloy, Go to your grafana and login with admin user.

1. Go to `Explore` page
2. In dropdown top left, select `Loki` data source
3. In the Query field, click on the `Label filters` button and select `namespace` -> select `lgtm` (or any other namespace where you have applications running)
4. Run query and you should see logs from your applications in the cluster.

### 7. (Optional) Install Node Exporter for node metrics

We will install Node Exporter to all nodes in the cluster to collect node metrics.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install node-exporter prometheus-community/prometheus-node-exporter -n lgtm
```

We have installed Prometheus with default value in the previous step, So it will automatically discover the Node Exporter and start collecting node metrics.

1. Go to your Grafana
2. Go to `Explore` page
3. Choose `Prometheus` data source
4. In query field, `node_memory_Active_bytes` and run the query, you should see the active memory usage of your nodes.

Import Grafana dashboard for node metrics:

1. Go to `Create` -> `Import`
2. In `Import via grafana.com` field, enter `1860` and click `Load`
3. Set dashboard name as desired, choose `Prometheus` data source and click `Import`

***You can also import other dashboards for Kubernetes metrics, such as `6417` for Kubernetes cluster monitoring, `315` for Kubernetes cluster monitoring (by Grafana Labs), `15757` for check the resources per pod/namespace, `11462` for monitoring traffic and requests, etc.***
