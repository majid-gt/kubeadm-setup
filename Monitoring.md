# Prometheus and Grafana Monitoring Setup in Kubernetes

## Overview

This document explains the monitoring architecture implemented in the Kubernetes cluster using Prometheus and Grafana. The monitoring stack is deployed using the kube-prometheus-stack Helm chart and managed through a GitOps workflow using ArgoCD.

The monitoring stack provides visibility into cluster health, node metrics, application metrics, and Kubernetes object state.

---

# 1. High-Level Architecture

The system architecture for application access and monitoring is structured as follows:

```
Internet
   │
   ▼
Domain DNS
(kcmkcmkcmkcmkcmkcmkcm.dpdns.org)
   │
   ▼
Ingress Controller (NGINX)
   │
   ├── /                → Frontend Service
   ├── /api             → Backend Service
   ├── /monitoring      → Grafana
   └── /prometheus      → Prometheus
```

Inside the Kubernetes cluster the monitoring stack is deployed as follows:

```
ArgoCD
   │
   ▼
Helm Chart (kube-prometheus-stack)
   │
   ├── Prometheus Operator
   │
   ├── Prometheus
   │   ├── scrapes nodes
   │   ├── scrapes pods
   │   ├── scrapes services
   │   └── stores metrics
   │
   ├── Alertmanager
   │
   ├── Grafana
   │
   ├── Node Exporter
   │
   └── kube-state-metrics
```

---

# 2. Why kube-prometheus-stack Was Used

Instead of installing monitoring components individually, the kube-prometheus-stack Helm chart was used.

This Helm chart installs the entire monitoring platform with a single deployment.

| Component | Purpose |
|-----------|--------|
| Prometheus Operator | Manages Prometheus instances |
| Prometheus | Collects and stores metrics |
| Alertmanager | Sends alerts |
| Grafana | Visualization dashboards |
| Node Exporter | Node-level metrics |
| kube-state-metrics | Kubernetes object metrics |

Using this chart simplifies deployment and ensures proper configuration between components.

---

# 3. GitOps Workflow

The cluster uses a GitOps model where all Kubernetes resources are stored in a Git repository and automatically deployed by ArgoCD.

Repository structure:

```
repo-root
├── argocd/
│   └── witnot-application.yml
│
├── k8s/
│   ├── monitoring/
│   │   ├── monitoring-app.yml
│   │   ├── namespace.yml
│   │   └── values.yml
│   │
│   ├── backend-deployment.yml
│   ├── frontend-deployment.yml
│   ├── hpa.yml
│   ├── ingress.yml
│   ├── namespace.yml
│   ├── postgres.yml
│   ├── redis.yml
│   └── services.yml
```

Deployment workflow:

```
Developer
     │
     │ git push
     ▼
GitHub Repository
     │
     ▼
ArgoCD watches repository
     │
     ▼
ArgoCD detects changes
     │
     ▼
ArgoCD deploys resources to cluster
```

In this model, Git acts as the single source of truth for the cluster configuration.

---

# 4. ArgoCD Deployment of Monitoring Stack

The monitoring stack is deployed through an ArgoCD Application resource.

File:

```
monitoring-app.yml
```

This file instructs ArgoCD to:

- Deploy the kube-prometheus-stack Helm chart
- Use custom values from the Git repository
- Deploy everything into the monitoring namespace

Multi-source deployment is used:

Source 1  
Helm chart repository

```
https://prometheus-community.github.io/helm-charts
```

Source 2  
Git repository containing custom configuration values.

Internally ArgoCD runs Helm similar to:

```
helm template kube-prometheus-stack
```

Using the custom configuration defined in:

```
values.yml
```

---

# 5. Purpose of values.yml

Helm charts are configurable using values files.

The values.yml file customizes the default configuration of the kube-prometheus-stack chart.

Example:

```
grafana:
  ingress:
    enabled: true
```

This configuration instructs Helm to create an Ingress resource for Grafana.

Another example:

```
grafana.ini:
  server:
    root_url: /monitoring
```

This allows Grafana to run behind the /monitoring subpath.

Without this configuration, accessing Grafana through:

```
/monitoring
```

would return a 404 error.

---

# 6. Kubernetes Resources Created

After Helm installation, multiple Kubernetes resources are created.

List all monitoring resources:

```
kubectl get all -n monitoring
```

Example resources created:

Pods

```
monitoring-stack-grafana
monitoring-stack-kube-prom-prometheus
monitoring-stack-alertmanager
monitoring-stack-node-exporter
monitoring-stack-kube-state-metrics
```

Services

```
grafana
prometheus
alertmanager
```

Ingress

```
grafana
prometheus
```

Custom Resource Definitions

```
servicemonitors
podmonitors
prometheusrules
```

---

# 7. Prometheus Operator

Prometheus Operator is a Kubernetes controller that introduces new resource types through Custom Resource Definitions.

Key CRDs:

```
ServiceMonitor
PodMonitor
PrometheusRule
```

List CRDs:

```
kubectl get crd
```

Example CRDs:

```
servicemonitors.monitoring.coreos.com
podmonitors.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
```

Prometheus automatically monitors these resources to discover targets.

---

# 8. Metrics Collection Pipeline

The monitoring pipeline operates as follows:

```
Node Exporter
      │
      ▼
Prometheus
      │
      ▼
Grafana
```

Node Exporter provides node-level metrics including:

- CPU usage
- Memory usage
- Disk usage
- Network usage

Metrics endpoint example:

```
http://node-exporter:9100/metrics
```

kube-state-metrics provides cluster state metrics such as:

- Pods
- Deployments
- ReplicaSets
- Nodes

Example metrics:

```
kube_pod_status_phase
kube_deployment_status_replicas
```

Prometheus discovers scrape targets through ServiceMonitor resources.

List ServiceMonitors:

```
kubectl get servicemonitors -n monitoring
```

---

# 9. Grafana Visualization

Grafana connects to Prometheus as a data source.

Data source verification:

Grafana → Settings → Data Sources

Expected source:

```
Prometheus
```

Dashboards query Prometheus using PromQL.

Example query:

```
rate(container_cpu_usage_seconds_total[5m])
```

Grafana converts query results into visual dashboards.

---

# 10. Request Flow for Monitoring Dashboard

Accessing Grafana through:

```
http://kcmkcmkcmkcmkcmkcmkcm.dpdns.org/monitoring
```

Request flow:

```
Browser
   │
   ▼
DNS Resolution
   │
   ▼
Ingress Controller
   │
   ▼
Ingress rule matches path /monitoring
   │
   ▼
Grafana Service
   │
   ▼
Grafana Pod
```

---

# 11. Grafana Subpath Configuration

Grafana typically expects to run at root path:

```
/
```

However in this architecture Grafana runs under:

```
/monitoring
```

Required configuration:

```
root_url
serve_from_sub_path
```

Without these settings Grafana would generate incorrect URLs.

---

# 12. CRD Annotation Error

During deployment an error occurred:

```
metadata.annotations too long
```

Cause:

Large CRD objects exceeded the Kubernetes annotation size limit when applied using client-side apply.

The annotation causing the issue:

```
kubectl.kubernetes.io/last-applied-configuration
```

Solution:

Use server-side apply or install CRDs using create:

```
kubectl create -f CRD.yaml
```

---

# 13. Monitoring Debugging Guide

Check pods:

```
kubectl get pods -n monitoring
```

Describe pod:

```
kubectl describe pod POD_NAME -n monitoring
```

Check services:

```
kubectl get svc -n monitoring
```

Check ingress:

```
kubectl get ingress -n monitoring
```

Check logs

Grafana logs:

```
kubectl logs -n monitoring deployment/monitoring-stack-grafana
```

Prometheus logs:

```
kubectl logs -n monitoring prometheus-monitoring-stack-kube-prom-prometheus-0
```

Check Prometheus targets by opening:

```
/prometheus
```

Navigate to:

```
Status → Targets
```

Expected targets:

```
node-exporter
kube-state-metrics
kubelet
```

---

# 14. ArgoCD Debugging

List ArgoCD applications:

```
kubectl get applications -n argocd
```

Describe monitoring application:

```
kubectl describe application monitoring-stack -n argocd
```

Manual synchronization:

```
argocd app sync monitoring-stack
```

---

# 15. Common Problems and Challenges

### Missing Prometheus CRDs

Error:

```
the server could not find the requested resource
(get prometheuses.monitoring.coreos.com)
```

Cause:

Prometheus CRDs were not installed.

Required CRDs:

```
prometheuses.monitoring.coreos.com
servicemonitors.monitoring.coreos.com
podmonitors.monitoring.coreos.com
alertmanagers.monitoring.coreos.com
```

Lesson: Install CRDs before deploying monitoring stack.

---

### CRD Annotation Size Error

Error:

```
metadata.annotations too long
```

Solution:

```
kubectl create -f CRD.yaml
```

instead of:

```
kubectl apply
```

---

### Prometheus Resource Missing

Issue:

```
kubectl get prometheus -n monitoring
```

returned no resources.

Cause:

ArgoCD skipped the resource when CRDs were missing.

Fix:

Manually create Prometheus resource.

---

### Grafana Plugin Error

Grafana displayed plugin errors because Prometheus was not running.

Lesson:

Ensure Prometheus is operational before troubleshooting Grafana.

---

### Dashboards Showing No Data

Cause:

Prometheus had no active scrape targets.

Verify:

Prometheus → Targets → all targets should show UP.

---

### NodePort Confusion

NodePort exposes services on a different port range.

Example:

```
port: 3000
nodePort: 30300
```

External access uses the NodePort.

---

### Manual Changes Outside Git

Manual resource creation breaks GitOps workflows.

Best practice:

All resources must be defined in Git repositories.

---

# 16. Complete Monitoring Setup Procedure

Step 1 Create monitoring namespace

```
kubectl create namespace monitoring
```

Step 2 Install Prometheus CRDs

```
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
```

```
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
```

```
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_podmonitors.yaml
```

```
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml
```

Step 3 Deploy Prometheus Operator

This operator manages:

- Prometheus
- Alertmanager
- ServiceMonitor
- PodMonitor

Step 4 Deploy kube-prometheus-stack

Installs:

- Prometheus
- Grafana
- Node Exporter
- kube-state-metrics
- Alertmanager

Step 5 Verify pods

```
kubectl get pods -n monitoring
```

Expected pods:

```
prometheus
grafana
alertmanager
node-exporter
kube-state-metrics
operator
```

Step 6 Verify Prometheus targets

Open Prometheus UI and check:

```
Status → Targets
```

All targets should be UP.

Step 7 Access Grafana

```
http://NODE_IP:NodePort
```

Default credentials:

```
admin / prom-operator
```

Step 8 Import dashboards

Recommended dashboards:

```
15757 – Kubernetes cluster monitoring
6417 – Kubernetes pods monitoring
1860 – Node exporter
```

Step 9 Filter namespace

Filter namespace to monitor application:

```
witnot
```

---

# 17. Monitoring Pipeline

Final monitoring pipeline:

```
Kubernetes Cluster
       │
       ▼
Node Exporter
Kube State Metrics
       │
       ▼
Prometheus
       │
       ▼
Grafana Dashboards
```

---

# 18. Production Improvements

Recommended improvements for production environments:

- Persistent storage for Prometheus
- Persistent storage for Grafana
- Alertmanager notifications
- Ingress configuration for Grafana and Prometheus
- Prometheus retention policies
- Alert rules for pod crashes
- Alert rules for CPU and memory thresholds

Prometheus can be integrated with alert channels such as:

- Slack
- Email
- PagerDuty
