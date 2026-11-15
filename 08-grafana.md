# Module 8 - Grafana

## Objective

In this module, we will learn how Grafana is used with Prometheus and Kubernetes to visualize metrics, create dashboards, and configure alert notifications.

We will use the Grafana instance deployed as part of the `kube-prometheus-stack` Helm chart.

By the end of this module, you will be able to:

- Understand Grafana and its role in monitoring
- Understand Grafana architecture
- Verify Grafana deployment in Kubernetes
- Access Grafana locally
- Log in to Grafana
- Understand the Grafana UI
- Configure Prometheus as a data source
- Explore Kubernetes metrics
- Create and understand dashboards
- Create dashboard panels
- Use PromQL inside Grafana
- Understand Grafana Alerting
- Create Grafana alert rules
- Configure Contact Points
- Configure Notification Policies
- Configure email notifications
- Configure SMTP securely
- Understand Grafana Alerting vs Prometheus Alerting
- Troubleshoot common Grafana problems

---

# 1. What is Grafana?

Grafana is an open-source observability and visualization platform.

It can connect to different data sources and display their data using:

- Graphs
- Time series
- Tables
- Stat panels
- Gauges
- Bar charts
- Logs
- Dashboards
- Alerts

In our project, Grafana is mainly used with Prometheus.

The basic architecture is:

```text
Kubernetes
    |
    | Metrics
    v
Prometheus
    |
    | PromQL
    v
Grafana
    |
    | Visualization
    v
Dashboard
```

Grafana does not normally collect Kubernetes metrics itself.

Prometheus collects and stores the metrics, while Grafana queries Prometheus and visualizes those metrics.

---

# 2. Grafana in Our Monitoring Stack

Our monitoring stack is based on:

```text
kube-prometheus-stack
```

The Helm release is:

```text
monitoring
```

The namespace is:

```text
monitoring
```

The chart version used in our environment is:

```text
87.4.0
```

---

# 3. Verify Grafana Deployment

Check the Grafana Pod:

```bash
kubectl get pods -n monitoring | grep grafana
```

Example:

```text
monitoring-grafana-67484bb798-7mrd7   3/3   Running
```

The important part is:

```text
3/3 Running
```

This indicates that all three containers in the Grafana Pod are ready.

Check all monitoring Pods:

```bash
kubectl get pods -n monitoring
```

---

# 4. Access Grafana Locally

Because Grafana is running inside Kubernetes, we can use Kubernetes port forwarding.

Get the Grafana Pod:

```bash
kubectl get pod \
-l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" \
-n monitoring
```

Port-forward:

```bash
kubectl port-forward \
-n monitoring \
pod/<grafana-pod-name> \
3000:3000
```

Example:

```bash
kubectl port-forward \
-n monitoring \
pod/monitoring-grafana-67484bb798-7mrd7 \
3000:3000
```

Grafana is then available at:

```text
http://localhost:3000
```

The connection is:

```text
Browser
   |
   | localhost:3000
   v
kubectl port-forward
   |
   v
Grafana Pod
   |
   v
Grafana
```

---

# 5. Grafana Login

The Grafana administrator username is:

```text
admin
```

The password is stored in a Kubernetes Secret.

Retrieve it using:

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" \
| base64 -d
```

The password should not be stored in Git.

---

# 6. Grafana UI

After logging in, Grafana provides areas such as:

```text
Home
Dashboards
Explore
Alerting
Connections
Administration
```

The exact UI layout can vary between Grafana versions.

The important concepts remain the same.

---

# 7. Grafana Data Sources

Grafana needs a data source from which it can retrieve metrics.

A data source is the connection between Grafana and the monitoring system that stores the data.

In our environment:

```text
Grafana
   |
   v
Prometheus
```

Prometheus is the primary data source for our Grafana dashboards.

---

# 8. Prometheus Data Source

The Prometheus data source allows Grafana to execute PromQL queries.

The workflow is:

```text
User
 |
 | PromQL
 v
Grafana
 |
 | Query
 v
Prometheus
 |
 | Time series data
 v
Grafana
 |
 v
Visualization
```

After configuring the Prometheus data source, use Grafana's data-source testing functionality to verify the connection.

---

# 9. Prometheus Service

Prometheus is exposed inside Kubernetes through a Service.

Check Services:

```bash
kubectl get svc -n monitoring
```

The general architecture is:

```text
Grafana
   |
   v
Prometheus Service
   |
   v
Prometheus Pod
```

---

# 10. Grafana Explore

Grafana provides an Explore interface for querying data interactively.

Explore is useful for:

- Testing PromQL
- Troubleshooting metrics
- Checking labels
- Verifying alert expressions
- Investigating Kubernetes behavior

Example:

```promql
up
```

Another example:

```promql
count(kube_pod_info)
```

---

# 11. Using PromQL in Grafana

Grafana can execute PromQL queries against Prometheus.

Example:

```promql
up
```

Example:

```promql
count(kube_pod_info)
```

Example:

```promql
rate(
  container_cpu_usage_seconds_total{
    namespace="default",
    pod=~"cpu-stress.*"
  }[2m]
)
```

---

# 12. Kubernetes Pod Metrics

Because kube-state-metrics is installed, Grafana can query Kubernetes object metrics.

Example:

```promql
kube_pod_info
```

Count Pods:

```promql
count(kube_pod_info)
```

---

# 13. Kubernetes Pod Status

Example:

```promql
kube_pod_status_ready
```

To investigate Pods whose Ready condition is false:

```promql
kube_pod_status_ready{
  condition="false"
}
```

---

# 14. Pod Restart Metrics

Kubernetes restart information can be queried using:

```promql
kube_pod_container_status_restarts_total
```

For example:

```promql
increase(
  kube_pod_container_status_restarts_total[5m]
)
```

This calculates the increase in the restart counter over the previous five minutes.

---

# 15. CPU Metrics

Container CPU usage can be investigated with:

```promql
rate(
  container_cpu_usage_seconds_total[5m]
)
```

For a specific workload:

```promql
rate(
  container_cpu_usage_seconds_total{
    namespace="default",
    pod=~"cpu-stress.*"
  }[2m]
)
```

---

# 16. Memory Metrics

Container memory usage can be investigated using:

```promql
container_memory_working_set_bytes
```

Example:

```promql
container_memory_working_set_bytes{
  namespace="default",
  pod=~"memory-stress.*"
}
```

---

# 17. Grafana Dashboards

A Grafana dashboard is a collection of visualizations.

A dashboard can contain multiple panels.

Conceptually:

```text
Dashboard
|
+-- CPU Panel
+-- Memory Panel
+-- Pod Count Panel
+-- Pod Restart Panel
+-- Node Disk Panel
+-- Kubernetes Status Panel
```

A dashboard provides a centralized view of system health.

---

# 18. Grafana Panels

A panel is an individual visualization inside a dashboard.

Examples:

```text
Time series
Stat
Gauge
Table
Bar chart
```

A panel generally contains:

```text
Data source
    +
Query
    +
Visualization
```

The workflow is:

```text
Prometheus
    |
    v
PromQL
    |
    v
Panel
    |
    v
Graph
```

---

# 19. Example Pod Count Panel

PromQL:

```promql
count(kube_pod_info)
```

A `Stat` visualization can be used to display the current Pod count.

---

# 20. Example CPU Panel

PromQL:

```promql
rate(
  container_cpu_usage_seconds_total{
    namespace="default"
  }[5m]
)
```

A time-series visualization can be used to observe CPU usage over time.

---

# 21. Example Memory Panel

PromQL:

```promql
container_memory_working_set_bytes{
  namespace="default"
}
```

The result can be displayed using a time-series visualization.

---

# 22. Dashboard Time Range

Grafana dashboards allow selecting a time range.

Examples:

```text
Last 5 minutes
Last 15 minutes
Last 1 hour
Last 6 hours
Last 24 hours
```

The selected time range affects the data shown by the panels.

---

# 23. Dashboard Refresh

Dashboards can be configured to refresh periodically.

Examples:

```text
5s
10s
30s
1m
5m
```

A shorter refresh interval provides more current information but can increase query frequency.

---

# 24. Grafana Variables

Grafana dashboard variables allow users to dynamically select values.

Instead of hard-coding:

```promql
namespace="default"
```

a dashboard can provide a namespace variable.

The query can then use the selected value.

Conceptually:

```text
Dashboard Variable
       |
       v
Selected Namespace
       |
       v
PromQL
       |
       v
Panel
```

Variables are useful for reusable Kubernetes dashboards.

---

# 25. Grafana Alerting

Grafana has its own alerting system.

Grafana Alerting allows alert rules to be created from queries and expressions.

The basic workflow is:

```text
Data Source
     |
     v
Query
     |
     v
Condition
     |
     v
Grafana Alert Rule
     |
     v
Contact Point
     |
     v
Notification Policy
     |
     v
Notification
```

This is different from Prometheus Alerting.

---

# 26. Grafana Alert Rule

A Grafana alert rule defines:

- Query
- Expression
- Condition
- Evaluation interval
- Pending period
- Labels
- Annotations

Conceptually:

```text
Prometheus Query
       |
       v
Grafana evaluates query
       |
       v
Condition
       |
       v
Alert Rule
```

---

# 27. Alert Rule Evaluation

Suppose a Grafana alert uses:

```promql
count(kube_pod_info)
```

Grafana periodically evaluates the query.

If the configured condition becomes true:

```text
Query
  |
  v
Condition true
  |
  v
Alert Pending
  |
  v
Pending duration
  |
  v
Alerting
```

---

# 28. Grafana Contact Point

A Contact Point defines where Grafana sends notifications.

Examples include:

```text
Email
Slack
Microsoft Teams
PagerDuty
Webhook
```

For our implementation, we used an email Contact Point.

A suitable name is:

```text
Gmail Notifications
```

or:

```text
Grafana Email
```

The important point is:

> The Contact Point represents the notification destination.

---

# 29. Contact Point Workflow

The relationship is:

```text
Grafana Alert Rule
       |
       v
Contact Point
       |
       v
Email
```

The Contact Point does not define the alert condition.

The alert rule defines the condition.

The Contact Point defines where the notification should go.

---

# 30. Notification Policy

A Notification Policy determines how Grafana routes alert notifications.

Conceptually:

```text
Grafana Alert
      |
      v
Notification Policy
      |
      v
Matching rules
      |
      v
Contact Point
      |
      v
Notification
```

A policy can route alerts based on labels such as:

```text
severity
namespace
team
environment
```

---

# 31. Alert Labels

Grafana alerts can contain labels.

Example:

```text
severity=warning
```

or:

```text
severity=critical
```

Labels are useful for routing and organizing alerts.

For example:

```text
severity=critical
        |
        v
Critical notification policy
        |
        v
Critical Contact Point
```

---

# 32. Email Notifications

Our Grafana alerting setup uses Gmail SMTP for email notifications.

The architecture is:

```text
Grafana Alert Rule
       |
       v
Notification Policy
       |
       v
Contact Point
       |
       v
Grafana SMTP
       |
       v
Gmail SMTP
       |
       v
Email
```

The SMTP server used is:

```text
smtp.gmail.com:587
```

Port `587` is used for SMTP submission with TLS.

---

# 33. Secure SMTP Configuration

The project uses an SMTP configuration file:

```text
grafana-smtp.yaml
```

The secure pattern is:

```yaml
grafana:
  grafana.ini:
    smtp:
      enabled: true
      host: smtp.gmail.com:587
      user: <YOUR_GMAIL_ADDRESS>
      password: $__env{GF_SMTP_PASSWORD}
      from_address: <YOUR_GMAIL_ADDRESS>
      from_name: Grafana
      skip_verify: false

  envFromSecret: grafana-smtp
```

The important security rule is:

```text
Do NOT put the real SMTP password in Git.
```

---

# 34. Environment Variable for SMTP Password

The configuration:

```yaml
password: $__env{GF_SMTP_PASSWORD}
```

means Grafana obtains the SMTP password through the environment variable:

```text
GF_SMTP_PASSWORD
```

The sensitive value is stored separately.

The workflow is:

```text
Kubernetes Secret
       |
       v
Environment Variable
       |
       v
Grafana
       |
       v
SMTP
```

---

# 35. SMTP Secret

The SMTP Secret should contain the sensitive value.

Conceptually:

```text
Secret
  |
  +-- GF_SMTP_PASSWORD
```

Grafana reads the Secret using:

```yaml
envFromSecret: grafana-smtp
```

The actual password should never be committed to Git.

---

# 36. Gmail App Password

For Gmail SMTP authentication, an App Password can be used when supported by the Google account configuration.

An App Password is different from the normal Gmail account password.

It should be treated as a secret.

Never commit an actual App Password to Git.

---

# 37. Bad and Good Secret Handling

Bad:

```yaml
password: "real-password"
```

Do not commit this.

Good:

```yaml
password: $__env{GF_SMTP_PASSWORD}
```

with the sensitive value stored separately in a Kubernetes Secret.

---

# 38. SMTP TLS

The configuration uses:

```yaml
skip_verify: false
```

This means TLS certificate verification is not disabled.

The SMTP configuration uses:

```text
smtp.gmail.com:587
```

with TLS.

---

# 39. Helm Upgrade for Grafana SMTP

The Grafana SMTP configuration was applied through Helm.

The command used in our environment was:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack   --version 87.4.0   -n monitoring   --reuse-values   -f grafana-smtp.yaml
```

The `--reuse-values` option keeps existing Helm values while applying the additional values file.

---

# 40. Helm Repository

The Prometheus Community repository was configured using:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update the repository:

```bash
helm repo update
```

Verify repositories:

```bash
helm repo list
```

---

# 41. Grafana SMTP Troubleshooting

If email notifications are not received, troubleshoot the workflow in layers:

```text
Grafana Alert Rule
       |
       v
Alert is firing
       |
       v
Notification Policy
       |
       v
Contact Point
       |
       v
SMTP configuration
       |
       v
SMTP authentication
       |
       v
Gmail
       |
       v
Email
```

An error at any layer can prevent the notification.

---

# 42. Verify Grafana Pod

Run:

```bash
kubectl get pods -n monitoring | grep grafana
```

The Pod should show:

```text
Running
```

and all containers should be ready.

---

# 43. Check Grafana Logs

If Grafana has an SMTP or alerting problem:

```bash
kubectl logs -n monitoring <grafana-pod-name>
```

Look for errors related to:

```text
SMTP
email
alert
notification
authentication
connection
```

---

# 44. Check Helm Values

The Helm values can be inspected with:

```bash
helm get values monitoring -n monitoring
```

Avoid exposing sensitive values in terminal screenshots or documentation.

---

# 45. Grafana vs Prometheus Alerting

There are two alerting paths in our monitoring environment.

## Prometheus Alerting

```text
Prometheus
    |
    v
PromQL
    |
    v
PrometheusRule
    |
    v
Prometheus
    |
    v
Alertmanager
    |
    v
Email
```

## Grafana Alerting

```text
Grafana
    |
    v
Query
    |
    v
Grafana Alert Rule
    |
    v
Notification Policy
    |
    v
Contact Point
    |
    v
Email
```

They are separate alerting mechanisms.

---

# 46. PrometheusRule vs Grafana Alert Rule

## PrometheusRule

Kubernetes Custom Resource:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
```

Managed by:

```text
Prometheus Operator
```

Typically evaluated by:

```text
Prometheus
```

Notifications are commonly handled by:

```text
Alertmanager
```

## Grafana Alert Rule

Created and managed inside Grafana.

It contains:

```text
Query
Condition
Evaluation
Labels
Annotations
```

Notifications are routed through:

```text
Notification Policy
       |
       v
Contact Point
```

---

# 47. Alerting Architecture Comparison

```text
                 Monitoring
                     |
          +----------+----------+
          |                     |
          v                     v
     Prometheus               Grafana
          |                     |
       PromQL                Query/PromQL
          |                     |
  PrometheusRule          Grafana Alert Rule
          |                     |
          v                     v
     Alertmanager        Notification Policy
          |                     |
          v                     v
        Email             Contact Point
```

Important interview point:

> Prometheus Alerting and Grafana Alerting are not the same thing.

---

# 48. When to Use Prometheus Alerting

Prometheus alerting is useful when:

- Alerts are infrastructure-oriented
- Kubernetes resources are monitored
- Alert rules should be stored as Kubernetes YAML
- Alert rules should be version controlled
- Alertmanager routing is required
- Prometheus-native alerting is preferred

Examples:

```text
Pod Not Ready
Pod Restart
OOMKilled
ImagePullBackOff
High CPU
High Memory
```

---

# 49. When to Use Grafana Alerting

Grafana Alerting is useful when:

- Alerting is closely tied to Grafana dashboards
- Multiple data sources are involved
- Teams want centralized alert management in Grafana
- Grafana notification routing is preferred
- Visualization and alerting are managed together

---

# 50. Avoid Duplicate Alerts

One important operational problem is duplicate notifications.

For example, the same CPU condition could be configured in both:

```text
Prometheus Alert
+
Grafana Alert
```

Both could send email notifications.

This can result in duplicate alerts.

Therefore, decide which system owns a particular alert.

A possible approach is:

```text
Kubernetes infrastructure alerts
        |
        v
Prometheus + Alertmanager

Dashboard/application-oriented alerts
        |
        v
Grafana Alerting
```

The exact strategy depends on the organization's architecture.

---

# 51. Grafana Alert Testing

A good Grafana alert test should verify:

```text
1. Query returns expected data
2. Alert condition is correct
3. Evaluation occurs
4. Alert becomes Pending
5. Alert becomes Alerting
6. Notification Policy matches
7. Contact Point receives the notification
8. Email is delivered
9. Alert resolves after the condition becomes false
```

---

# 52. Grafana Alert Lifecycle

A simplified lifecycle is:

```text
Normal
  |
  v
Condition becomes true
  |
  v
Pending
  |
  v
Alerting
  |
  v
Notification
  |
  v
Condition becomes false
  |
  v
Normal / Recovered
```

The exact state names can depend on Grafana version and alert-rule configuration.

---

# 53. Grafana Troubleshooting Checklist

When Grafana does not show metrics:

```text
[ ] Grafana Pod is Running
[ ] Prometheus Pod is Running
[ ] Prometheus Service exists
[ ] Prometheus data source is configured
[ ] Data source connection succeeds
[ ] PromQL returns data
```

When Grafana alerting does not work:

```text
[ ] Query returns data
[ ] Alert condition is correct
[ ] Alert rule is enabled
[ ] Evaluation interval is correct
[ ] Alert becomes firing/alerting
[ ] Notification Policy matches
[ ] Contact Point is correct
[ ] SMTP is configured
[ ] SMTP credentials are valid
[ ] Grafana logs show no notification error
```

---

# 54. Useful Kubernetes Commands

Check Grafana:

```bash
kubectl get pods -n monitoring | grep grafana
```

Check all monitoring Pods:

```bash
kubectl get pods -n monitoring
```

Check monitoring Services:

```bash
kubectl get svc -n monitoring
```

Check Grafana logs:

```bash
kubectl logs -n monitoring <grafana-pod-name>
```

Port-forward Grafana:

```bash
kubectl port-forward -n monitoring pod/<grafana-pod-name> 3000:3000
```

---

# 55. Useful Helm Commands

List releases:

```bash
helm list -A
```

List repositories:

```bash
helm repo list
```

Update repositories:

```bash
helm repo update
```

View Helm values:

```bash
helm get values monitoring -n monitoring
```

View release status:

```bash
helm status monitoring -n monitoring
```

---

# 56. Security Best Practices

Never commit:

```text
Gmail password
Gmail App Password
API token
Access token
Database password
Private key
Kubernetes Secret values
```

Use:

```text
Kubernetes Secrets
Environment variables
External secret-management systems
```

instead.

For this project, the Grafana SMTP configuration intentionally uses:

```yaml
password: $__env{GF_SMTP_PASSWORD}
```

instead of storing the real password in the YAML file.

---

# 57. Git Security Check

Before committing Grafana configuration, search for sensitive values.

Example:

```bash
grep -RniE "password|passwd|token|api[_-]?key" .
```

Review the results carefully.

A placeholder such as:

```text
<GMAIL_APP_PASSWORD>
```

is different from a real credential.

Never replace the placeholder with the real secret before committing.

---

# 58. Grafana Architecture Summary

The complete Grafana monitoring path is:

```text
Kubernetes
     |
     | Metrics
     v
Prometheus
     |
     | PromQL
     v
Grafana
     |
     +---- Explore
     |
     +---- Dashboards
     |
     +---- Panels
     |
     +---- Alert Rules
               |
               v
       Notification Policy
               |
               v
         Contact Point
               |
               v
             SMTP
               |
               v
             Email
```

---

# 59. Project Implementation Summary

In this project we:

```text
1. Installed kube-prometheus-stack
2. Deployed Grafana
3. Verified the Grafana Pod
4. Accessed Grafana through port-forwarding
5. Logged into Grafana
6. Used Prometheus as the metrics data source
7. Queried Kubernetes metrics using PromQL
8. Worked with dashboards and panels
9. Worked with Grafana Alerting
10. Configured a Contact Point
11. Configured notification routing
12. Configured Gmail SMTP
13. Used a secure environment-variable approach for the SMTP password
```

---

# 60. Important Interview Questions

## What is Grafana?

Grafana is an observability and visualization platform used to query, visualize, and monitor data from different data sources.

## Does Grafana collect metrics?

Normally, Grafana does not act as the primary metrics collector in this setup.

Prometheus collects and stores metrics.

Grafana queries Prometheus and visualizes the results.

## What is a Grafana dashboard?

A dashboard is a collection of panels used to visualize and monitor data.

## What is a Grafana panel?

A panel is an individual visualization containing a query and visualization configuration.

## What is a Grafana data source?

A data source is a configured connection from Grafana to a system that provides data.

In our project, Prometheus is the main data source.

## What is Grafana Alerting?

Grafana Alerting is Grafana's built-in alerting system for evaluating conditions and routing notifications.

## What is a Contact Point?

A Contact Point defines where Grafana sends alert notifications.

Examples include email, Slack, PagerDuty, and webhooks.

## What is a Notification Policy?

A Notification Policy determines how Grafana routes alerts to Contact Points.

## What is the difference between Contact Point and Notification Policy?

Contact Point:

```text
Where to send
```

Notification Policy:

```text
Which alerts should go there
```

## What is the difference between Grafana Alerting and Prometheus Alerting?

Prometheus Alerting typically follows:

```text
PrometheusRule
    |
    v
Prometheus
    |
    v
Alertmanager
```

Grafana Alerting follows:

```text
Grafana Alert Rule
    |
    v
Notification Policy
    |
    v
Contact Point
```

## Why use SMTP?

SMTP allows Grafana to send email notifications through an SMTP server.

In this project:

```text
smtp.gmail.com:587
```

is used.

## Why should the SMTP password not be stored in Git?

Because credentials committed to Git can be exposed to anyone with repository access and may remain in Git history even after deletion.

Secrets should be stored separately.

---

# 61. Key Takeaways

The most important concepts from this module are:

```text
Grafana
   |
   v
Visualization

Prometheus
   |
   v
Metrics

PromQL
   |
   v
Queries

Dashboard
   |
   v
Collection of panels

Panel
   |
   v
Individual visualization

Grafana Alert Rule
   |
   v
Alert condition

Notification Policy
   |
   v
Routing

Contact Point
   |
   v
Notification destination

SMTP
   |
   v
Email delivery
```

The complete monitoring architecture is:

```text
                    Kubernetes
                        |
                        v
               Metrics / Events
                        |
          +-------------+-------------+
          |                           |
          v                           v
   kube-state-metrics            Node Exporter
          |                           |
          +-------------+-------------+
                        |
                        v
                   Prometheus
                        |
             +----------+----------+
             |                     |
             v                     v
       Prometheus Alerting      Grafana
             |                     |
             v                     |
        Alertmanager          PromQL / UI
             |                     |
             v                     +------> Dashboards
           Email                   |
                                   +------> Grafana Alerting
                                               |
                                               v
                                      Notification Policy
                                               |
                                               v
                                         Contact Point
                                               |
                                               v
                                              Email
```

The key lesson is:

> Prometheus is responsible for collecting and storing metrics, while Grafana provides visualization and can also provide its own alerting and notification workflow.

---

# End of Module 8
