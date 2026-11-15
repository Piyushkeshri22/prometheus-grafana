# Module 7 - Kubernetes Alert Rules

## Objective

In this module, we implement and document Kubernetes alert rules using PrometheusRule resources.

The alert rules in this project cover CPU usage, memory usage, disk usage, ImagePullBackOff, OOMKilled containers, Pod Not Ready conditions, and pod restarts.

We also created practical Kubernetes manifests to deliberately generate several failure conditions and verify that Prometheus detects them.

By the end of this module, you will be able to:

- Create Kubernetes alert rules using `PrometheusRule`
- Write PromQL expressions for Kubernetes alerts
- Configure warning and critical severity levels
- Use `for` durations to avoid transient alerts
- Use labels and annotations
- Detect high CPU and memory usage relative to resource limits
- Detect high node filesystem usage
- Detect ImagePullBackOff
- Detect OOMKilled containers
- Detect Pods that are not Ready
- Detect frequent container restarts
- Create Prometheus recording rules
- Create practical Kubernetes failure scenarios for testing
- Understand the complete alert testing workflow

---

## Prerequisites

Before starting this module, the following components should be available:

- Kubernetes Cluster
- Prometheus
- Prometheus Operator
- Alertmanager
- kube-state-metrics
- Node Exporter
- PromQL knowledge

The alerting concepts used in this module are covered in:

```text
06-prometheus-alerting.md
```

---

## Alert Rules Implemented

The project contains the following main rule files:

| File | Purpose |
|---|---|
| `cpu-high-alert.yml` | Detects high CPU usage |
| `memory-high-alert.yml` | Detects high memory usage |
| `disk-high-alert.yml` | Detects high node filesystem usage |
| `image-pull-alert.yml` | Detects `ImagePullBackOff` |
| `oom-killed-alert.yml` | Detects containers terminated by `OOMKilled` |
| `pod-not-ready-alert.yml` | Detects Pods that are not Ready |
| `pod-restart-alert.yml` | Detects frequent container restarts |
| `recording-rules.yml` | Creates a reusable Pod count time series |
| `test-alert.yml` | Synthetic alert used for Prometheus alerting testing |

---

## Practical Test Manifests

The project also contains practical Kubernetes manifests:

```text
alert-rules/practical/
├── broken-pod.yml
├── cpu-stress.yml
├── memory-stress.yml
├── not-ready-pod.yml
├── oom-demo.yml
└── restart-demo.yml
```

These manifests deliberately create conditions that can be detected by the corresponding alert rules.

---

# PrometheusRule Structure

All main alert files use:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
```

A typical structure is:

```text
PrometheusRule
    ↓
metadata
    ↓
spec
    ↓
groups
    ↓
rules
    ↓
alert
expr
for
labels
annotations
```

The rules use:

```yaml
labels:
  release: monitoring
```

to associate them with the `monitoring` Prometheus installation managed by the Prometheus Operator.

---

# Alert Severity

This project uses two main severity levels:

```text
warning
critical
```

Example:

```yaml
labels:
  severity: warning
```

and:

```yaml
labels:
  severity: critical
```

Warning alerts indicate an important condition that should be investigated.

Critical alerts indicate a more serious condition requiring faster attention.

---

# 1. Test Alert

File:

```text
alert-rules/test-alert.yml
```

The alert is:

```yaml
- alert: MyFirstPrometheusAlert
  expr: vector(0) == 1
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "My first Prometheus alert"
    description: "This alert is created for Prometheus testing."
```

The expression:

```promql
vector(0) == 1
```

creates a synthetic condition. Since `vector(0)` produces `0`, this expression is normally false.

The purpose of this rule is to provide a simple Prometheus alerting test without depending on a real Kubernetes failure.

---

# 2. CPU High Alert

File:

```text
alert-rules/cpu-high-alert.yml
```

The file contains two alert levels:

| Alert | Threshold | `for` | Severity |
|---|---:|---:|---|
| `KubernetesPodHighCPUUsageWarning` | > 80% of CPU limit | 5m | warning |
| `KubernetesPodHighCPUUsageCritical` | > 90% of CPU limit | 2m | critical |

## CPU PromQL

The core calculation is:

```promql
(
  rate(
    container_cpu_usage_seconds_total{
      namespace="default",
      pod=~"cpu-stress.*"
    }[2m]
  )
  /
  on(namespace, pod)
  kube_pod_container_resource_limits{
    namespace="default",
    pod=~"cpu-stress.*",
    resource="cpu"
  }
) * 100
```

Conceptually:

```text
CPU usage
    ÷
CPU limit
    × 100
    =
CPU usage percentage of the limit
```

`rate()` is used because CPU usage is derived from a cumulative CPU time metric.

The `on(namespace, pod)` clause matches the CPU usage series with the CPU limit series using the `namespace` and `pod` labels.

## CPU Practical Test

File:

```text
alert-rules/practical/cpu-stress.yml
```

The manifest creates a Deployment named:

```text
cpu-stress
```

The container runs:

```yaml
command:
  - /bin/sh
  - -c
  - "while true; do :; done"
```

This creates a continuous CPU workload.

Resources:

```yaml
requests:
  cpu: 50m
  memory: 16Mi

limits:
  cpu: 200m
  memory: 32Mi
```

The alert rule selects Pods matching:

```promql
pod=~"cpu-stress.*"
```

so the practical workload matches the alert selector.

---

# 3. Memory High Alert

File:

```text
alert-rules/memory-high-alert.yml
```

The file contains two severity levels:

| Alert | Threshold | `for` | Severity |
|---|---:|---:|---|
| `KubernetesPodHighMemoryUsageWarning` | > 80% of memory limit | 3m | warning |
| `KubernetesPodHighMemoryUsageCritical` | > 90% of memory limit | 1m | critical |

## Memory PromQL

```promql
(
  container_memory_working_set_bytes{
    namespace="default",
    pod=~"memory-stress.*"
  }
  /
  on(namespace,pod)
  kube_pod_container_resource_limits{
    namespace="default",
    pod=~"memory-stress.*",
    resource="memory"
  }
) * 100
```

Conceptually:

```text
Memory working set
       ÷
Memory limit
       × 100
       =
Memory usage percentage of the limit
```

## Memory Practical Test

File:

```text
alert-rules/practical/memory-stress.yml
```

The Deployment is named:

```text
memory-stress
```

It uses:

```text
polinux/stress
```

Resources:

```yaml
requests:
  memory: "100Mi"

limits:
  memory: "200Mi"
```

The stress workload uses:

```yaml
--vm
1
--vm-bytes
150M
--vm-hang
0
```

This provides a practical memory-pressure scenario against the configured memory limit.

---

# 4. Disk Usage Alert

File:

```text
alert-rules/disk-high-alert.yml
```

The disk alert contains two levels:

| Alert | Threshold | `for` | Severity |
|---|---:|---:|---|
| `KubernetesNodeDiskUsageWarning` | > 80% | 5m | warning |
| `KubernetesNodeDiskUsageCritical` | > 90% | 2m | critical |

## Disk PromQL

```promql
(
  (
    node_filesystem_size_bytes{fstype!~"tmpfs|ramfs"}
    -
    node_filesystem_avail_bytes{fstype!~"tmpfs|ramfs"}
  )
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|ramfs"}
) * 100
```

Conceptually:

```text
Disk used
    =
Disk size - Available disk

Disk usage %
    =
(Disk size - Available disk)
÷
Disk size
× 100
```

The expression excludes:

```text
tmpfs
ramfs
```

using:

```promql
fstype!~"tmpfs|ramfs"
```

## Disk Practical Testing

A dedicated disk-stress Kubernetes manifest was not created in this project.

Therefore:

```text
Disk alert rule       ✅ configured
Dedicated disk test   ❌ not created
```

We do not claim a disk practical test that was not performed.

---

# 5. ImagePullBackOff Alert

File:

```text
alert-rules/image-pull-alert.yml
```

Alert:

```yaml
alert: KubernetesPodImagePullBackOff
```

Expression:

```promql
kube_pod_container_status_waiting_reason{
  reason="ImagePullBackOff"
} == 1
```

Configuration:

```yaml
for: 1m

labels:
  severity: warning
```

## Image Pull Practical Test

File:

```text
alert-rules/practical/broken-pod.yml
```

The Pod intentionally uses an invalid image:

```yaml
image: nginx:this-tag-does-not-exist
```

Workflow:

```text
Invalid image
      ↓
Kubernetes cannot pull image
      ↓
Container enters waiting state
      ↓
ImagePullBackOff
      ↓
Prometheus metric changes
      ↓
Alert expression becomes true
      ↓
Pending
      ↓
Firing
```

---

# 6. OOMKilled Alert

File:

```text
alert-rules/oom-killed-alert.yml
```

Alert:

```yaml
alert: KubernetesPodOOMKilled
```

Expression:

```promql
kube_pod_container_status_last_terminated_reason{
  reason="OOMKilled",
  namespace!~"kube-system|monitoring"
} == 1
```

The rule excludes:

```text
kube-system
monitoring
```

Configuration:

```yaml
for: 1m

labels:
  severity: warning
```

## OOM Practical Test

File:

```text
alert-rules/practical/oom-demo.yml
```

Memory limit:

```yaml
limits:
  memory: "64Mi"
```

Stress workload:

```yaml
--vm
1
--vm-bytes
128M
--vm-hang
1
```

The intended scenario is:

```text
Memory limit = 64Mi
        ↓
Stress workload attempts 128M
        ↓
Container exceeds memory limit
        ↓
Container is OOMKilled
        ↓
kube_pod_container_status_last_terminated_reason
        ↓
reason="OOMKilled"
        ↓
Alert condition becomes true
```

---

# 7. Pod Not Ready Alert

File:

```text
alert-rules/pod-not-ready-alert.yml
```

Alert:

```yaml
alert: KubernetesPodNotReady
```

Expression:

```promql
kube_pod_status_ready{
  condition="false",
  namespace!~"kube-system|monitoring"
} == 1
```

Configuration:

```yaml
for: 1m

labels:
  severity: warning
```

## Pod Not Ready Practical Test

File:

```text
alert-rules/practical/not-ready-pod.yml
```

Pod:

```text
not-ready-demo
```

Readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 80

  initialDelaySeconds: 5
  periodSeconds: 5
```

The `/health` endpoint is not provided by the default Nginx configuration used here, so the readiness check can fail.

Workflow:

```text
Readiness probe fails
        ↓
Pod Ready condition becomes false
        ↓
kube_pod_status_ready
        ↓
condition="false"
        ↓
PromQL expression becomes true
        ↓
Pending
        ↓
Firing
```

---

# 8. Pod Restart Alert

File:

```text
alert-rules/pod-restart-alert.yml
```

Two alerts are configured:

| Alert | Condition | `for` | Severity |
|---|---|---:|---|
| `KubernetesPodRestartWarning` | > 1 restart in 5m | 2m | warning |
| `KubernetesPodRestartCritical` | > 5 restarts in 5m | 1m | critical |

## Restart PromQL

```promql
increase(
  kube_pod_container_status_restarts_total{
    namespace!~"kube-system|monitoring"
  }[5m]
)
```

This calculates the increase in the container restart counter over the last five minutes.

Warning:

```promql
increase(...) > 1
```

Critical:

```promql
increase(...) > 5
```

## Restart Practical Test

File:

```text
alert-rules/practical/restart-demo.yml
```

The container executes:

```yaml
echo "Starting..."
sleep 5
exit 1
```

and uses:

```yaml
restartPolicy: Always
```

The cycle becomes:

```text
Container starts
      ↓
Wait 5 seconds
      ↓
exit 1
      ↓
Kubernetes restarts container
      ↓
Repeat
```

This produces an increasing restart counter.

---

# 9. Recording Rule

File:

```text
alert-rules/recording-rules.yml
```

Recording rule:

```yaml
- record: cluster:pod_count
```

Expression:

```promql
count(kube_pod_info)
```

Evaluation interval:

```yaml
interval: 30s
```

Prometheus evaluates the expression every 30 seconds and stores the result under:

```text
cluster:pod_count
```

Instead of repeatedly calculating:

```promql
count(kube_pod_info)
```

we can query:

```promql
cluster:pod_count
```

Conceptually:

```text
Original PromQL
      ↓
Recording Rule
      ↓
cluster:pod_count
      ↓
Grafana / Prometheus queries
```

---

# Alert-to-Test Mapping

| Practical Manifest | Generated Condition | Related Alert |
|---|---|---|
| `cpu-stress.yml` | High CPU usage | CPU High |
| `memory-stress.yml` | High memory usage | Memory High |
| `not-ready-pod.yml` | Readiness failure | Pod Not Ready |
| `oom-demo.yml` | OOMKilled | OOMKilled |
| `restart-demo.yml` | Repeated container exits | Pod Restart |
| `broken-pod.yml` | ImagePullBackOff | Image Pull Failure |
| No dedicated manifest | High filesystem usage | Disk High |

---

# Complete Alert Testing Workflow

The practical workflow is:

```text
1. Create or apply a Kubernetes test manifest
              ↓
2. Kubernetes creates the failure condition
              ↓
3. Exporter / kube-state-metrics exposes the condition
              ↓
4. Prometheus scrapes the metric
              ↓
5. Prometheus evaluates the PromQL expression
              ↓
6. Condition becomes true
              ↓
7. Alert enters Pending
              ↓
8. `for` duration is completed
              ↓
9. Alert becomes Firing
              ↓
10. Alertmanager receives the alert
              ↓
11. Alertmanager processes the alert
              ↓
12. Notification is sent
```

When the problem is fixed:

```text
Problem fixed
      ↓
Metric returns to normal
      ↓
PromQL expression becomes false
      ↓
Alert becomes Resolved
      ↓
Resolved notification can be sent
```

---

# Alert Lifecycle Example

For the Pod Not Ready alert:

```text
not-ready-demo
      ↓
Readiness probe fails
      ↓
Pod Not Ready
      ↓
Prometheus expression = true
      ↓
Pending
      ↓
1 minute
      ↓
Firing
      ↓
Alertmanager
      ↓
Notification
```

After the Pod becomes Ready:

```text
Pod becomes Ready
      ↓
PromQL expression = false
      ↓
Resolved
```

---

# Severity Strategy

The project uses different thresholds and durations for warning and critical conditions.

Example CPU:

```text
> 80%
    ↓
warning
    ↓
for 5m
```

and:

```text
> 90%
    ↓
critical
    ↓
for 2m
```

The same approach is used for CPU, Memory, Disk, and Pod restarts.

Other alerts, such as OOMKilled, ImagePullBackOff, and Pod Not Ready, use warning severity in the current configuration.

---

# Troubleshooting Alert Rules

## Step 1 — Test the PromQL

Run the expression directly in Prometheus.

Example:

```promql
kube_pod_status_ready{
  condition="false",
  namespace!~"kube-system|monitoring"
} == 1
```

If the expression returns no series, the alert condition is not currently true or the metric/filter does not match the available data.

---

## Step 2 — Verify the PrometheusRule

Run:

```bash
kubectl get prometheusrules -A
```

Verify that the rule exists.

---

## Step 3 — Check Prometheus Rules

Check the Prometheus UI and verify that the alert rule has been loaded.

Determine whether the alert is:

```text
Inactive
Pending
Firing
```

---

## Step 4 — Check the Practical Workload

Verify that the corresponding resource exists.

For example:

```bash
kubectl get pods
```

Examples:

```text
cpu-stress
memory-stress
oom-demo
not-ready-demo
restart-demo
broken-nginx
```

---

## Step 5 — Check the Underlying Metric

If an alert does not fire, query the underlying metric directly.

Examples:

```promql
kube_pod_status_ready
```

```promql
kube_pod_container_status_restarts_total
```

```promql
kube_pod_container_status_last_terminated_reason
```

This helps determine whether the issue is with:

- Kubernetes condition
- Metric
- Label selector
- PromQL expression
- Alert rule

---

# Common Problems

| Problem | Possible Cause |
|---|---|
| Alert does not fire | Condition is not true |
| Alert stays Pending | `for` duration not completed |
| No data | Wrong metric or labels |
| Rule missing | PrometheusRule not loaded |
| CPU alert has no data | Pod name/labels do not match |
| Memory alert has no data | Resource-limit metric does not match |
| Restart alert does not fire | Restart increase is below threshold |
| Image alert does not fire | Pod is not actually in `ImagePullBackOff` |
| OOM alert does not fire | Container was not OOMKilled |
| Pod Not Ready alert does not fire | Pod readiness condition is true |

---

# Practical Validation Checklist

For each applicable alert, verify:

```text
[ ] PrometheusRule exists
[ ] PromQL expression works
[ ] Required metric exists
[ ] Labels match
[ ] Practical workload exists
[ ] Failure condition is generated
[ ] Alert enters Pending
[ ] Alert becomes Firing
[ ] Alertmanager receives it
[ ] Notification is generated
[ ] Condition is fixed
[ ] Alert becomes Resolved
```

Not every alert in this project has a dedicated practical manifest, so the checklist should be applied according to the test scenario available for each rule.

---

# Best Practices

## 1. Use meaningful alert names

Examples:

```text
KubernetesPodNotReady
KubernetesPodOOMKilled
KubernetesPodRestartCritical
KubernetesPodHighCPUUsageCritical
```

Alert names should clearly describe the condition.

---

## 2. Use appropriate thresholds

Avoid firing alerts for every small temporary change.

Use realistic thresholds and suitable `for` durations.

---

## 3. Use severity labels

Example:

```yaml
labels:
  severity: critical
```

This allows Alertmanager or other systems to route alerts based on severity.

---

## 4. Use useful annotations

Example:

```yaml
annotations:
  summary: "Pod Not Ready"
  description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is not ready."
```

This makes notifications useful to the person receiving them.

---

## 5. Match test workloads with alert selectors

For example, the CPU alert uses:

```promql
pod=~"cpu-stress.*"
```

and the practical workload is named:

```text
cpu-stress
```

Keeping selectors and test resources consistent makes testing easier.

---

## 6. Avoid inventing test results

If a rule has not been practically tested, document it as configured rather than claiming it was tested.

For example, this project has a disk alert rule but no dedicated disk-stress manifest.

---

# Repository Structure

The alerting implementation is organized as:

```text
alert-rules/
├── cpu-high-alert.yml
├── disk-high-alert.yml
├── image-pull-alert.yml
├── memory-high-alert.yml
├── oom-killed-alert.yml
├── pod-not-ready-alert.yml
├── pod-restart-alert.yml
├── recording-rules.yml
├── test-alert.yml
└── practical/
    ├── broken-pod.yml
    ├── cpu-stress.yml
    ├── memory-stress.yml
    ├── not-ready-pod.yml
    ├── oom-demo.yml
    └── restart-demo.yml
```

The documentation explains the purpose and behavior of these files, while the YAML files contain the actual implementation.

---

# Key Takeaways

The Kubernetes alerting implementation in this project follows:

```text
Kubernetes condition
       ↓
Metric
       ↓
PromQL
       ↓
PrometheusRule
       ↓
Pending
       ↓
Firing
       ↓
Alertmanager
       ↓
Notification
       ↓
Resolved
```

The project demonstrates several types of Kubernetes alerting:

```text
Resource Usage
    ├── CPU
    ├── Memory
    └── Disk

Pod State
    ├── Not Ready
    ├── ImagePullBackOff
    └── OOMKilled

Pod Behavior
    └── Frequent Restarts

Prometheus Features
    ├── Alerting Rules
    └── Recording Rules
```

The main practical lesson is:

> A useful alerting implementation is not just a PromQL expression. It combines a meaningful condition, an appropriate threshold, a suitable duration, useful labels and annotations, and a practical way to validate that the alert actually fires.

The next module will focus on the Grafana installation and configuration used in this project.
