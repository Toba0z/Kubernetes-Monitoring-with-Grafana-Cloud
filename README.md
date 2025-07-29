# Kubernetes Monitoring with Grafana Cloud

## Table of Contents

- [Introduction](#introduction)
- [Architecture Diagram](#architecture-diagram)
- [Setting Up Grafana Cloud](#setting-up-grafana-cloud)
  - [Create Access Policies](#create-access-policies)
  - [Configure Terraform Provider](#configure-terraform-provider)
  - [Create a Grafana Stack and Service Account](#create-a-grafana-stack-and-service-account)
  - [Create Access Policy for Grafana Alloy](#create-access-policy-for-grafana-alloy)
  - [Expose Connection Details](#expose-connection-details)
- [Deploying the Monitoring Stack](#deploying-the-monitoring-stack)
  - [Deploying with Terraform](#deploying-with-terraform)
  - [What the k8s-monitoring Helm Chart Deploys](#what-the-k8s-monitoring-helm-chart-deploys)
- [Understanding Grafana Alloy](#understanding-grafana-alloy)
- [Customizing Your Monitoring Pipeline](#customizing-your-monitoring-pipeline)
- [Managing Costs](#managing-costs)
- [Resources](#resources)

## Introduction

This guide walks you through building a scalable and centralized monitoring system for Kubernetes environments using a push-based model — all managed through Infrastructure as Code.

The solution is made up of three core components:

1. **Grafana Cloud** — a powerful observability platform that serves as the central hub for storing and visualizing telemetry data.
2. **Grafana Alloy** (previously known as Grafana Agent) — a versatile agent designed to automatically detect, collect, process, and forward data such as metrics, logs, and traces from various sources.

<p align="center">
  <img src="https://grafana.com/media/blog/grafana-alloy-launch/grafana-alloy-architecture-diagram-2.png?w=1920" alt="Kubernetes Monitoring Architecture">
  <br/>
  <sub><i>Diagram courtesy of the Grafana blog — illustrating Grafana Alloy as an OpenTelemetry collector with integrated Prometheus pipelines.</i></sub>
</p>

3. **k8s-monitoring Helm chart** — a pre-configured setup for deploying monitoring components in Kubernetes clusters, simplifying the deployment and configuration of Grafana Alloy.

With this architecture, you can collect logs, metrics, traces, and more across multiple Kubernetes clusters and centralize everything in one place.

![Kubernetes Monitoring Architecture](/image-2.webp)

<sub><i>This diagram illustrates how metrics are gathered from applications within a Kubernetes cluster and then forwarded to Grafana Cloud or a self-hosted alternative for centralized observability.</i></sub>


## Setting Up Grafana Cloud

To begin, we’ll configure Grafana Cloud so it can receive and process monitoring data from our Kubernetes clusters.

### Create Access Policies

Start by setting up access policies in Grafana Cloud. These policies control which actions specific tools — like Terraform — are allowed to perform on your Grafana Cloud resources.

1. If you haven’t already, [sign up for Grafana Cloud](https://grafana.com/products/cloud/).
2. Navigate to **My Account** → **Security** → **Access Policies**.
3. Create a new policy named `terraform-access-policy`.
4. Assign it the necessary permissions so Terraform can create, update, and manage resources in your Grafana Cloud environment:

![Kubernetes Monitoring Architecture](/IMAGE-3.webp)
<sub><i>For production use, you should define more specific permissions</i></sub>

## Configure Terraform Provider

Next, we’ll configure the Grafana Terraform provider so it can communicate with your Grafana Cloud account.

```h
provider "grafana" {
  alias                     = "cloud"
  cloud_access_policy_token = var.grafana_cloud_access_policy_token
}
```

## Create a Grafana Stack

A Grafana Stack is a logical collection of observability services — such as metrics, logs, and traces — grouped under one configuration. Let’s create one using Terraform.

````h
resource "grafana_cloud_stack" "this" {
  name        = "Your name for Grafana Cloud Stack"
  slug        = "yourgcloudstack"
  region_slug = "us"
}
 ````

## Create a Service Account for the Stack
You’ll also need a service account to manage the stack via Terraform:

```h
resource "grafana_cloud_stack_service_account" "cloud_sa" {
  provider = grafana.cloud

  stack_slug  = resource.grafana_cloud_stack.this[0].slug
  name        = "cloud service account"
  role        = "Admin"
  is_disabled = false
}
```

## Generate a Token for the Service Account
Finally, create a token so the service account can authenticate:

```h
resource "grafana_cloud_stack_service_account_token" "cloud_sa" {
  provider = grafana.cloud

  stack_slug         = resource.grafana_cloud_stack.this[0].slug
  name               = "terraform service account key"
  service_account_id = grafana_cloud_stack_service_account.cloud_sa[0].id
}
```

## Create Access Policy for Grafana Alloy

To enable Grafana Alloy to push observability data to your Grafana Cloud stack, you’ll need to create an access policy that grants the required permissions. This includes write access for metrics, logs, and traces.

### Define the Access Policy

The following Terraform resource creates an access policy with the necessary scopes for Grafana Alloy:

```h
resource "grafana_cloud_access_policy" "grafana_alloy" {
  provider = grafana.cloud

  region       = resource.grafana_cloud_stack.this[0].region_slug
  name         = "grafana-alloy-access-policy"
  display_name = "Grafana Alloy Access Policy (created by Terraform)"

  scopes = [
    "metrics:write",
    "metrics:import",
    "logs:write",
    "traces:write",
  ]

  # Specify the stack this policy applies to
  realm {
    type       = "stack"
    identifier = resource.grafana_cloud_stack.this[0].id
  }
}
```
## Generate an Access Token for Alloy
Once the policy is in place, use this resource to create a token that Grafana Alloy can use to authenticate and push data:
```h
resource "grafana_cloud_access_policy_token" "grafana_alloy" {
  provider = grafana.cloud

  region           = resource.grafana_cloud_stack.this[0].region_slug
  access_policy_id = grafana_cloud_access_policy.grafana_alloy[0].policy_id
  name             = "grafana-alloy-access-token"
  display_name     = "Grafana Alloy Access Token (created by Terraform)"
}

✅ This ensures Grafana Alloy has the correct access to send telemetry data to your stack securely.

```
## Expose Connection Details

At this stage, we’ve provisioned all the necessary tokens and endpoints that Grafana Alloy needs to start pushing telemetry data into Grafana Cloud.

The following Terraform outputs make it easier to retrieve these details for use in your Kubernetes and Alloy configurations:

```h
output "alloy_access_token" {
  description = "Grafana Alloy access token."
  value       = grafana_cloud_access_policy_token.grafana_alloy[0].token
  sensitive   = true
}

output "prometheus_url" {
  description = "Grafana Cloud Prometheus URL."
  value       = resource.grafana_cloud_stack.this[0].prometheus_url
}

output "prometheus_user_id" {
  description = "Grafana Cloud Prometheus user ID."
  value       = resource.grafana_cloud_stack.this[0].prometheus_user_id
}

output "loki_url" {
  description = "Grafana Cloud Loki URL."
  value       = resource.grafana_cloud_stack.this[0].logs_url
}

output "loki_user_id" {
  description = "Grafana Cloud Loki user ID."
  value       = resource.grafana_cloud_stack.this[0].logs_user_id
}
```
<sub><i>This Terraform configuration defines outputs for Prometheus (Mimir) and Loki, providing user IDs and endpoint URLs. While Grafana Cloud also supports Tempo for tracing, k6 for load testing, and more, this setup focuses solely on metrics and logs to keep things simple.</i></sub>

## Deploy the `k8s-monitoring` Helm Chart

Now that Grafana Cloud is ready, the next step is to deploy monitoring components into your Kubernetes cluster using the `k8s-monitoring` Helm chart.

> ⚠️ **Note:** To deploy Helm charts via Terraform, you'll first need to configure the [Terraform Helm provider](https://registry.terraform.io/providers/hashicorp/helm/latest/docs). Be sure to set it up before continuing.

This example uses Terraform for simplicity and to demonstrate how infrastructure can be managed declaratively. However, for production environments, consider alternatives like **Kustomize** or **ArgoCD**. These tools provide better separation of concerns and more fine-grained control over the lifecycle of Kubernetes resources.

> 📌 Helm via Terraform is ideal for quick setups, but GitOps tools like ArgoCD may be better suited for managing large-scale or multi-cluster deployments in the long run.

## Understanding Grafana Alloy

Grafana Alloy (formerly Grafana Agent) is configured using **River**, a purpose-built configuration language inspired by HCL (used in Terraform). It allows you to define how telemetry data — such as metrics and logs — is discovered, scraped, relabeled, and pushed to Grafana Cloud.

While it’s possible to write Alloy configurations manually as shown below, this approach can quickly become difficult to manage in production environments. For more scalable workflows, it's recommended to use automation tools or Helm with templated values.

---

### 🟢 Prometheus Metrics Collection

The following River configuration demonstrates how Alloy discovers Kubernetes pods, relabels them, scrapes metrics, and sends them to Grafana Cloud via remote write:

```h
// Discover Kubernetes pods to collect metrics
discovery.kubernetes "pod_metrics" {
  role = "pod"
  namespaces {
    own_namespace = false
    names         = ["default", "kube-system", "monitoring", "celery"]
  }
}

// Relabel pod metadata for metric labeling
discovery.relabel "pod_metrics" {
  targets = discovery.kubernetes.pod_metrics.targets

  rule {
    action = "labelmap"
    regex  = "__meta_kubernetes_pod_label_(.+)"
  }
  rule {
    action        = "replace"
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
  rule {
    action      = "replace"
    replacement = "${cluster_name}"
    target_label = "cluster"
  }
}

// Scrape metrics from discovered pods
prometheus.scrape "pod_metrics" {
  targets       = discovery.relabel.pod_metrics.output
  forward_to    = [prometheus.remote_write.default.receiver]
  honor_labels  = true
}

// Send metrics to Prometheus remote_write endpoint
prometheus.remote_write "default" {
  endpoint {
    url = "${prometheus_url}"
    basic_auth {
      username = "${prometheus_user_id}"
      password = "${grafana_agent_access_token}"
    }
  }
}
```
## 🔵 Kubernetes Logs Collection
Similarly, this configuration collects logs from specific namespaces, applies relabeling rules, and sends the log data to Grafana Loki:
```h
// Discover pods for log collection
discovery.kubernetes "pod_logs" {
  role = "pod"
  namespaces {
    own_namespace = false
    names         = ["celery"]
  }
}

// Relabel log sources for enrichment
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pod_logs.targets

  rule {
    action = "labelmap"
    regex  = "__meta_kubernetes_pod_label_(.+)"
  }
  rule {
    action        = "replace"
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "__host__"
  }
  rule {
    action        = "replace"
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    action        = "replace"
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    action        = "replace"
    source_labels = ["__meta_kubernetes_container_name"]
    target_label  = "container"
  }
  rule {
    action        = "replace"
    replacement   = "/var/log/pods/*$1/*.log"
    separator     = "/"
    source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
    target_label  = "__path__"
  }
}

// Tail logs from Kubernetes containers
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.write.default.receiver]
}

// Forward logs to Grafana Loki
loki.write "default" {
  endpoint {
    url = "${loki_url}"
    basic_auth {
      username = "${loki_user_id}"
      password = "${grafana_agent_access_token}"
    }
  }
}
```
<sub><i>A telemetry pipeline using Grafana Alloy that automatically discovers Kubernetes pods, transforms metadata through relabeling, and routes metrics (Prometheus) and logs (Loki) to Grafana Cloud endpoints.</i></sub>

## Deploying Grafana Alloy with Terraform

To roll out Grafana Alloy in your Kubernetes cluster using Terraform and Helm, follow the configuration below. This setup uses a Helm chart to deploy the agent and injects the River configuration dynamically.

### Step 1: Create a Monitoring Namespace

Define a namespace for monitoring components:

```hcl
resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}
```
## Step 2: Render the River Config Template
Use Terraform's templatefile() function to populate the River config with your cluster-specific values and Grafana Cloud credentials:
```h
locals {
  grafana_agent_config = templatefile("${path.module}/config/grafana_agent_config.river.tpl", {
    cluster_name                = "your_cluster"
    grafana_agent_access_token = var.grafana_agent_access_token
    prometheus_url             = var.prometheus_url
    prometheus_user_id         = var.prometheus_user_id
    loki_url                   = var.loki_url
    loki_user_id               = var.loki_user_id
  })
}

📁 This assumes you’ve created a grafana_agent_config.river.tpl file in a config/ directory alongside your Terraform module.
```

## Step 3: Deploy the Grafana Agent using Helm
Install the Grafana Alloy Helm chart and inject your generated config:

```h
resource "helm_release" "grafana_agent" {
  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana-agent"
  version    = "~> 0.29.0"
  name       = "grafana-agent"
  namespace  = kubernetes_namespace.monitoring.metadata[0].name

  values = [yamlencode({
    agent = {
      configMap = {
        content = local.grafana_agent_config
      }
    }
  })]
}

```

## Why Use the `k8s-monitoring` Helm Chart?

Manually writing and maintaining detailed telemetry configurations for Grafana Alloy can quickly become repetitive and error-prone — especially as your environment scales.

To simplify deployment and ensure best practices are followed, we’ll use the [`k8s-monitoring` Helm chart](https://github.com/grafana/k8s-monitoring) maintained by Grafana.

---

## What the `k8s-monitoring` Helm Chart Deploys

This chart bundles a comprehensive monitoring stack for Kubernetes, including:

- **Prometheus** — for metrics collection.
- **Loki** — for log aggregation.
- **Tempo** — for distributed tracing.
- **Pyroscope** — for continuous profiling.

It also includes core Kubernetes monitoring components:

- **kube-state-metrics** — exposes the state of Kubernetes objects.
- **node-exporter** — gathers hardware metrics from nodes.
- **kubelet** — exposes node and container runtime metrics.
- **cAdvisor** — captures container-level CPU, memory, network, and disk usage.

> 🔍 *For the full list of components and values, refer to the [GitHub repository](https://github.com/grafana/k8s-monitoring).*

---

## Example Configuration Snippet

Here's an example of how the chart might be configured to forward metrics and logs to Grafana Cloud:

```yaml
cluster:
  name: ${cluster_name}

externalServices:
  prometheus:
    host: "${prometheus_host}"
    basicAuth:
      username: "${prometheus_username}"
      password: "${grafana_agent_access_token}"

  loki:
    host: "${loki_host}"
    basicAuth:
      username: "${loki_username}"
      password: "${grafana_agent_access_token}"

metrics:
  enabled: true
  scrapeInterval: 60s
  autoDiscover:
    enabled: true
    annotations:
      scrape: "k8s.grafana.com/scrape"
      metricsPath: "k8s.grafana.com/metrics.path"
      metricsPortNumber: "k8s.grafana.com/metrics.portNumber"

  alloy:
    enabled: false
  kube-state-metrics:
    enabled: true
  node-exporter:
    enabled: true
  kubelet:
    enabled: true
  cadvisor:
    enabled: true
  apiserver:
    enabled: false
  kubeControllerManager:
    enabled: false
  kubeProxy:
    enabled: false
  kubeScheduler:
    enabled: false
  cost:
    enabled: false

logs:
  enabled: true
  pod_logs:
    enabled: true
    discovery: "all"
    annotation: "k8s.grafana.com/logs.autogather"
  cluster_events:
    enabled: false

traces:
  enabled: false

prometheus-node-exporter:
  enabled: true
  tolerations:
    - effect: NoSchedule
      operator: Exists
    - effect: NoExecute
      operator: Exists

alloy-logs:
  controller:
    type: daemonset
    nodeSelector:
      kubernetes.io/os: linux
    tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists

```
---
## Deploying the `k8s-monitoring` Helm Chart with Terraform

Once your configuration values are set, you can deploy the full monitoring stack to your Kubernetes cluster using Terraform. This approach ensures your observability setup is repeatable and version-controlled.

### Terraform Deployment

The following Terraform code deploys the `k8s-monitoring` chart into a dedicated `monitoring` namespace and injects values from a templated YAML file:

```h
resource "kubernetes_namespace" "monitoring" {
  metadata {
    name = "monitoring"
  }
}

resource "helm_release" "k8s_monitoring" {
  repository = "https://grafana.github.io/helm-charts"
  chart      = "k8s-monitoring"
  version    = "~> 1.6.13"

  name      = "k8s-monitoring"
  namespace = kubernetes_namespace.monitoring.metadata[0].name

  values = [
    templatefile("${path.module}/config/k8s_monitoring.yaml.tpl", {
      cluster_name                 = "your_cluster"
      grafana_access_policy_token = var.grafana_access_policy_token
      grafana_prometheus_host     = var.grafana_prometheus_host
      grafana_prometheus_username = var.grafana_prometheus_username
      grafana_loki_host           = var.grafana_loki_host
      grafana_loki_username       = var.grafana_loki_username
    })
  ]
}
```
## Verifying the Deployment
Once deployed, check the status of the pods in the monitoring namespace:

```
$ kubectl get pods -n monitoring
```
You should see output similar to:

```sql
NAME                                                 READY   STATUS    RESTARTS   AGE
k8s-monitoring-alloy-0                               2/2     Running   0          4d2h
k8s-monitoring-alloy-logs-6qgz9                      2/2     Running   0          4d2h
k8s-monitoring-alloy-logs-8mpqd                      2/2     Running   0          4d1h
k8s-monitoring-alloy-logs-gdj2b                      2/2     Running   0          4d2h
k8s-monitoring-alloy-logs-ljg9b                      2/2     Running   0          4d2h
k8s-monitoring-alloy-logs-tnn4d                      2/2     Running   0          4d1h
k8s-monitoring-alloy-logs-vglxx                      2/2     Running   0          4d2h
k8s-monitoring-alloy-logs-vrldw                      2/2     Running   0          4d1h
k8s-monitoring-kube-state-metrics-55f4b6bdd7-pswx5   1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-54nhd        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-nfhv4        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-pbh6l        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-qj86w        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-rlhmj        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-tpdz9        1/1     Running   0          4d2h
k8s-monitoring-prometheus-node-exporter-wwm7s        1/1     Running   0          4d2h
```
<sub><i>Monitoring components deployed in the <code>monitoring</code> namespace, including Grafana Alloy for metrics and logs, kube-state-metrics for cluster state, and node-exporter for hardware metrics.</i></sub>

This confirms that your monitoring infrastructure — including Grafana Alloy (metrics & logs), kube-state-metrics, and node-exporter — is up and running in the monitoring namespace.

## Customizing Your Monitoring Pipeline

In real-world scenarios, especially across multiple environments or Kubernetes clusters, you may need to tailor how metrics are discovered, labeled, and forwarded.

The `k8s-monitoring` Helm chart makes this easier by allowing you to inject custom relabeling logic — without having to manually write full River configurations.

### Adding Extra Relabeling Rules

You can enhance your observability by appending custom relabeling rules directly in your Helm values:

```yaml
metrics:
  # Rule blocks to extend discovery.relabel logic for all metric sources
  # Reference: https://grafana.com/docs/agent/latest/flow/reference/components/discovery.relabel/#rule-block
  extraRelabelingRules: |-
    rule {
      action = "replace"
      source_labels = ["__meta_kubernetes_pod_node_name"]
      target_label = "node"
    }
    rule {  // Tag environment of pods in GPU clusters
      action = "replace"
      source_labels = ["__meta_kubernetes_pod_label_Env"]
      target_label = "env"
    }
```
<sub><i>These enhancements provide deeper context in your Grafana dashboards without modifying your application code or pod annotations.</i></sub>

This configuration adds:

A node label that reflects which Kubernetes node a pod is running on.
An env label that distinguishes environments, particularly helpful in GPU or multi-tenant clusters.

These relabeling rules use the same syntax as River, but are embedded directly within the managed Helm chart deployment. This approach offers the flexibility and power of River-based customization — without the complexity of writing and maintaining standalone configurations — making your monitoring setup both scalable and maintainable.

## Managing Costs

Monitoring large-scale Kubernetes environments in the cloud can lead to significant data ingestion and storage costs — especially when collecting logs and metrics from every node, container, and namespace.

Here are a few strategies to optimize your monitoring footprint and stay cost-effective:

---

### ✅ Start Small

Begin by enabling monitoring only for the most critical components and namespaces. Avoid turning on logs or metrics for everything by default. Focus on high-value workloads first.

---

### 📊 Monitor Your Usage

Regularly check Grafana Cloud’s usage dashboards to see:

- How much telemetry data is being ingested
- Which metrics are contributing the most to your usage
- Where you might be overspending on non-critical observability

---

### 🔍 Filter Unnecessary Metrics

Use the `metricsTuning` configuration built into the `k8s-monitoring` Helm chart to control what metrics get collected and stored.

#### Kube State Metrics (KSM)

```yaml
kube-state-metrics:
  metricsTuning:
    useDefaultAllowList: true
    includeMetrics:
      - kube_pod_status_qos_class
      - kube_namespace_created
      - kube_deployment_status_replicas_unavailable
      - kube_pod_container_status_restarts_total
      - kube_node_labels
    excludeMetrics:
      - kube_lease_owner
      - kube_lease_renew_time
      - kube_pod_tolerations
      - kube_pod_status_ready
      - kube_pod_status_scheduled
      - kube_pod_owner
      # ... (truncated for brevity)

 ### Node Exporter
node-exporter:
  metricsTuning:
    useDefaultAllowList: true
    useIntegrationAllowList: false
    includeMetrics:
      - node_uname_info
      - node_cpu_core_throttles_total
      - node_network_receive_bytes_total
      # ...
    excludeMetrics:
      - node_filesystem_readonly
      - node_scrape_collector_success
      - node_cpu_guest_seconds_total

## Kubelet Metrics
kubelet:
  metricsTuning:
    useDefaultAllowList: true
    includeMetrics: []
    excludeMetrics:
      - kubelet_pod_worker_duration_seconds_bucket
      - storage_operation_duration_seconds_count
      - volume_manager_total_volumes

## cAdvisor Metrics
cadvisor:
  metricsTuning:
    useDefaultAllowList: true
    includeMetrics:
      - machine_cpu_cores
      - container_cpu_cfs_throttled_seconds_total
    excludeMetrics:
      - container_memory_cache
      - container_fs_reads_total
      - container_fs_writes_bytes_total

```
### 🎯 Limit Log Collection by Namespace
To avoid excessive log volume, restrict log collection to only the namespaces you actually need:

```yaml
logs:
  pod_logs:
    namespaces:
      - celery
      - supabase

##💡 Fine-tuning what you collect ensures that your observability stack remains both actionable and affordable.
```



