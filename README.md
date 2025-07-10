# ✅ Calico Setup with Felix, Typha & Kube Controllers Monitoring in Grafana


---

## 🔧 Prerequisites

* Kubernetes Cluster (tested on EKS, Minikube, and Kubeadm)
* Prometheus + Grafana already deployed (via Helm)
* Helm installed
* Cluster admin access

# Prometheus

```
export PROM_POD=$(kubectl get pods -n observability -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward -n observability $PROM_POD 9090:9090
```

# Grafana

```
export GRAFANA_POD=$(kubectl get pods -n observability -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward -n observability $GRAFANA_POD 3000:3000
```

---

## 🧱 Step 1: Install Calico with Tigera Operator

### 🔹 1.1 Add the Calico Helm repo

```bash
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm repo update
```

### 🔹 1.2 Install Tigera Operator

```bash
helm upgrade --install calico projectcalico/tigera-operator \
  --namespace tigera-operator \
  --create-namespace
```

### 🔹 1.3 Apply Calico Installation CRD

Create `calico-installation.yaml`:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

Apply:

```bash
kubectl apply -f calico-installation.yaml
```

---

## 📈 Step 2: Enable Metrics for Felix, Typha, and Controllers

### 🔹 2.1 Configure Felix Metrics

Create or patch FelixConfiguration:

```bash
kubectl apply -f - <<EOF
apiVersion: crd.projectcalico.org/v1
kind: FelixConfiguration
metadata:
  name: default
spec:
  prometheusMetricsEnabled: true
  prometheusMetricsPort: 9091
EOF
```

### 🔹 2.2 Enable Typha Metrics

Patch the Typha Deployment to enable metrics:

```bash
kubectl -n calico-system set env deployment/calico-typha \
  TYPHA_PROMETHEUSMETRICSENABLED=true \
  TYPHA_PROMETHEUSMETRICSPORT=9093
```

### 🔹 2.3 Enable Kube Controllers Metrics

Patch the kube-controllers deployment:

```bash
kubectl -n calico-system set env deployment/calico-kube-controllers \
  ENABLED_CONTROLLERS=workloadendpoint,namespace,policy,serviceaccount,profile,node \
  PROMETHEUS_METRICS_ENABLED=true \
  PROMETHEUS_METRICS_PORT=9094
```

---

## 📡 Step 3: Expose Metrics via Kubernetes Services

Create `calico-metrics-services.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: calico-felix-metrics
  namespace: calico-system
  labels:
    app: calico-felix
spec:
  selector:
    k8s-app: calico-node
  ports:
  - name: http-metrics
    port: 9091
    targetPort: 9091
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: calico-typha-metrics
  namespace: calico-system
  labels:
    app: calico-typha
spec:
  selector:
    k8s-app: calico-typha
  ports:
  - name: http-metrics
    port: 9093
    targetPort: 9093
  clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: calico-kube-controllers-metrics
  namespace: calico-system
  labels:
    app: calico-kube-controllers
spec:
  selector:
    k8s-app: calico-kube-controllers
  ports:
  - name: http-metrics
    port: 9094
    targetPort: 9094
  clusterIP: None
```

Apply it:

```bash
kubectl apply -f calico-metrics-services.yaml
```

---

## 📊 Step 4: Configure Prometheus Scraping

Add the following to your Prometheus `values.yaml` (or patch `prometheus.yaml` under `additionalScrapeConfigs`):

```yaml
- job_name: calico-felix
  static_configs:
  - targets:
    - calico-felix-metrics.calico-system.svc.cluster.local:9091

- job_name: calico-typha
  static_configs:
  - targets:
    - calico-typha-metrics.calico-system.svc.cluster.local:9093

- job_name: calico-kube-controllers
  static_configs:
  - targets:
    - calico-kube-controllers-metrics.calico-system.svc.cluster.local:9094
```

Then upgrade Prometheus:

```bash
helm upgrade -i prometheus prometheus-community/prometheus \
  -f _config/prometheus.yaml -n observability
```

---

## 📉 Step 5: Import Grafana Calico Dashboards

### Option 1: Import from Grafana.com

* Dashboard ID: `12023` (Kubernetes Calico)
* Or use ID `3244` (Project Calico Felix)

### Option 2: Import from Local JSON

If you have a local JSON:

* Go to Grafana → Dashboards → Import
* Upload the JSON file
* Set data source to `Prometheus`

---

## ✅ Validation

After setup:

* Check `calico-node`, `calico-typha`, and `calico-kube-controllers` pods are Running
* In Prometheus UI, query:

  * `felix_active_local_endpoints`
  * `typha_broadcast_sent_messages`
  * `calico_kube_controllers_policy_total`
* Confirm Grafana panels are populated

---

## 🧼 (Optional) Cleanup Dummy Tagged Endpoint

If you created a test workload endpoint for `felix_active_local_tags`, remove it:

```bash
calicoctl delete wep <name>
```

---

## 🎉 Done!

You’ve now set up a full Calico monitoring pipeline from scratch:

* Calico components installed and running
* Prometheus scraping Felix, Typha, and kube-controllers
* Grafana showing metrics dashboard
