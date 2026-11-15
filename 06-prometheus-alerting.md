# Module 6 - Prometheus Alerting**

**## Objective**

In this module, we will learn how Prometheus detects infrastructure problems and generates alerts based on predefined conditions.

We will understand the Prometheus alerting architecture, alert rules, PrometheusRule, alert lifecycle, Alertmanager, routing, receivers, grouping, and notification handling.

The practical Kubernetes alert rules created during this project will be documented separately in Module 7.

By the end of this module, you will be able to:

- Understand Prometheus Alerting
- Understand Monitoring vs Alerting
- Understand the Prometheus Alerting Architecture
- Understand PrometheusRule
- Create alert expressions using PromQL
- Understand \`for\`
- Understand alert labels and annotations
- Understand the alert lifecycle
- Understand Pending, Firing, and Resolved states
- Understand Recording Rules
- Understand Alertmanager
- Understand Alertmanager routing
- Understand receivers
- Understand alert grouping
- Configure email notifications
- Troubleshoot alerting problems

**---**

**# Prerequisites**

Before starting this module, ensure the following components are available:

- Kubernetes Cluster
- Prometheus
- Prometheus Operator
- Alertmanager
- kube-state-metrics
- Node Exporter
- PromQL knowledge

PromQL fundamentals are covered in:

\`\`\`text
05-promql-fundamentals.md
## Topics Covered
## What is Monitoring?
## What is Alerting?
## Monitoring vs Alerting
## Why Alerting is Required
## Prometheus Alerting Architecture
PrometheusRule
Alert Rule Structure
### expr
### for
## Labels
## Annotations
## Alert Lifecycle
Inactive State
Pending State
Firing State
Resolved State
## Recording Rules
Alertmanager
Alertmanager Configuration
Routes
Receivers
Grouping
Notification Flow
Gmail Notification Architecture
Alert Troubleshooting
## Practical Alerting Workflow
Best Practices
## Key Takeaways
## What is Monitoring?

Monitoring is the process of continuously collecting and observing system metrics to understand the current health and performance of infrastructure.

For example, monitoring can tell us:

Current CPU usage
Current memory usage
Number of running pods
Number of pod restarts
Network traffic
Node health
Kubernetes resource status

Example:

CPU Usage:       45%
Memory Usage:    62%
Running Pods:    18
Node Status:     Healthy

Monitoring helps us understand what is happening in the infrastructure.

## What is Alerting?

Alerting is the process of automatically notifying engineers when a predefined condition becomes true.

Instead of continuously checking dashboards manually, an alerting system can detect problems and notify the responsible team.

Examples:

CPU usage is greater than 80%
Memory usage is greater than 80%
A pod is not ready
A pod is restarting frequently
A container was OOMKilled
An image cannot be pulled
A node becomes unavailable

Example:

CPU Usage > 80%
        ↓
Alert condition becomes true
        ↓
Prometheus evaluates the rule
        ↓
Alert is generated
## Monitoring vs Alerting
| Monitoring | Alerting |
|---|---|
| Shows system state | Notifies about problems |
| Usually dashboard-oriented | Notification-oriented |
| Helps investigate | Helps detect |
| Requires engineers to observe | Automatically notifies |
| Example: CPU = 85% | Example: CPU > 80% for 5 minutes |

Both monitoring and alerting are important.

Monitoring helps us understand the system.

Alerting helps us respond to important problems.

## Why Alerting is Required

Without alerting, engineers may have to continuously check dashboards.

For example:

Pod crashes
     ↓
Nobody checks dashboard
     ↓
Problem remains unnoticed

With alerting:

Pod crashes
     ↓
Prometheus detects condition
     ↓
Alert fires
     ↓
Notification sent
     ↓
Engineer investigates

Benefits include:

Faster incident detection
Reduced downtime
Automated notifications
Faster troubleshooting
Improved reliability
Better production monitoring
## Prometheus Alerting Architecture

The basic Prometheus alerting architecture is:

Kubernetes / Application
          ↓
       Metrics
          ↓
Exporters / kube-state-metrics
          ↓
      Prometheus
          ↓
    Alert Rule Evaluation
          ↓
      Alert Generated
          ↓
      Alertmanager
          ↓
   Notification Receiver
          ↓
 Email / Slack / Teams / PagerDuty

In our Kubernetes environment:

Kubernetes
     ↓
kube-state-metrics
     ↓
Prometheus
     ↓
PrometheusRule
     ↓
Alertmanager
     ↓
Notification
## Components Used
| Component | Purpose |
|---|---|
| Prometheus | Collects metrics and evaluates alert rules |
| PromQL | Defines the condition for an alert |
| PrometheusRule | Kubernetes resource used to define rules |
| Prometheus Operator | Manages Prometheus and PrometheusRule resources |
| Alertmanager | Handles and routes alerts |
| kube-state-metrics | Provides Kubernetes object metrics |
| Node Exporter | Provides node-level metrics |
| Receiver | Defines where notifications are sent |

## What is a PrometheusRule?

Prometheus Operator provides a Kubernetes Custom Resource called:

PrometheusRule

A PrometheusRule allows us to define recording rules and alerting rules as Kubernetes YAML resources.

Instead of manually configuring alert rules inside a Prometheus configuration file, we can create a Kubernetes resource.

Basic structure:

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: example-alert

spec:
  groups:
    - name: example.rules

      rules:

        - alert: ExampleAlert
          expr: <PromQL expression>
          for: 1m

          labels:
            severity: warning

          annotations:
            summary: Example alert
            description: Example description
## PrometheusRule Structure

Important fields include:

### apiVersion
### kind
### metadata
### spec
### groups
### rules
### alert
### expr
### for
labels
annotations
### apiVersion

For Prometheus Operator alert rules:

apiVersion: monitoring.coreos.com/v1

This specifies the API version of the PrometheusRule resource.

### kind
kind: PrometheusRule

This tells Kubernetes that we are creating a PrometheusRule resource.

### metadata

Example:

metadata:
  name: pod-not-ready-alert

Metadata identifies the Kubernetes resource.

Other metadata fields can include:

namespace:
labels:
annotations:
### spec

The spec section contains the Prometheus rule configuration.

Example:

spec:
  groups:
    - name: kubernetes.rules
### groups

Alert rules are organized into groups.

Example:

groups:
  - name: kubernetes.rules

A group can contain multiple related rules.

Example:

kubernetes.rules
      │
      ├── Pod Not Ready
      ├── Pod Restart
      ├── OOMKilled
      └── Image Pull Failure
### rules

The rules section contains individual alerting or recording rules.

Example:

rules:

  - alert: KubernetesPodNotReady
    expr: kube_pod_status_ready{condition="false"} == 1
### alert

The alert field defines the name of the alert.

Example:

alert: KubernetesPodNotReady

The alert name is displayed in Prometheus, Alertmanager, Grafana, and notification messages.

Good alert names should be:

Descriptive
Consistent
Easy to understand

Examples:

KubernetesPodNotReady
KubernetesPodRestarting
KubernetesPodOOMKilled
KubernetesImagePullFailure
### expr

The expr field contains the PromQL expression used to determine whether the alert condition is true.

Example:

expr: kube_pod_status_ready{condition="false"} == 1

The expression is evaluated continuously by Prometheus.

If the expression returns a matching result, the alert condition becomes active.

## Example Alert Expression

Consider:

kube_pod_status_ready{condition="false"} == 1

The expression identifies pods whose Ready condition is false.

Conceptually:

Pod Ready = false
        ↓
PromQL expression returns result
        ↓
Alert condition becomes active
### for

The for field specifies how long the alert condition must remain true before the alert enters the Firing state.

Example:

for: 1m

This means:

Condition becomes true
        ↓
## Pending
        ↓
Wait 1 minute
        ↓
Condition still true
        ↓
## Firing

The for duration helps prevent temporary conditions from immediately generating alerts.

Example of for

Suppose the rule is:

expr: kube_pod_status_ready{condition="false"} == 1

for: 1m

If the pod becomes Not Ready for only 20 seconds:

Not Ready
   ↓
## Pending
   ↓
Pod becomes Ready
   ↓
Alert does not fire

If the pod remains Not Ready for more than one minute:

Not Ready
   ↓
## Pending
   ↓
1 minute passes
   ↓
## Firing
## Labels

Labels provide additional information about an alert.

Example:

labels:
  severity: warning

Common labels include:

severity
team
environment
category
namespace

Labels can be used by Alertmanager for routing and grouping.

Example:

labels:
  severity: critical
  category: kubernetes
## Annotations

Annotations provide descriptive information about the alert.

Example:

annotations:
  summary: Kubernetes pod is not ready

  description: >
    A Kubernetes pod has been in a Not Ready state
    for more than one minute.

Annotations are useful in notification messages because they explain what happened.

## Labels vs Annotations
| Labels | Annotations |
|---|---|
| Identify/classify alerts | Describe alerts |
| Used for routing | Used for notification content |
| Used for grouping | Provide human-readable information |
| Example: `severity=warning` | Example: `summary=Pod not ready` |

## Complete Alert Rule Example

Example:

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule

metadata:
  name: pod-not-ready-alert

spec:
  groups:
    - name: kubernetes.rules

      rules:

        - alert: KubernetesPodNotReady

          expr: kube_pod_status_ready{condition="false"} == 1

          for: 1m

          labels:
            severity: warning
            category: kubernetes

          annotations:
            summary: Kubernetes pod is not ready
            description: >
              A Kubernetes pod has been in a Not Ready
              state for more than one minute.

This is the basic structure we used later for our practical Kubernetes alerting.

## Alert Lifecycle

Prometheus alerts normally move through different states.

The important states are:

## Inactive
   ↓
## Pending
   ↓
## Firing
   ↓
## Resolved
## Inactive

The alert condition is false.

Example:

Pod Ready = true

The alert is inactive.

No notification is generated.

## Pending

The alert expression has become true, but the configured for duration has not completed.

Example:

for: 1m

Flow:

Condition becomes true
        ↓
## Pending
        ↓
Waiting for 1 minute
## Firing

The condition remains true for the configured for duration.

Example:

Pod remains Not Ready
        ↓
## Pending
        ↓
1 minute completed
        ↓
## Firing

The firing alert can then be handled by Alertmanager.

## Resolved

When the alert condition becomes false again, the alert is resolved.

Example:

Pod Not Ready
     ↓
## Firing
     ↓
Pod becomes Ready
     ↓
## Resolved

If send_resolved is enabled in the notification configuration, a resolved notification can also be sent.

## Complete Alert Lifecycle Example
Pod becomes Not Ready
        ↓
Prometheus evaluates rule
        ↓
Condition = true
        ↓
## Pending
        ↓
Condition remains true
        ↓
## Firing
        ↓
Alertmanager
        ↓
Notification
        ↓
Problem fixed
        ↓
Condition = false
        ↓
## Resolved

This lifecycle was later tested with our Kubernetes Pod Not Ready alert.

## Recording Rules

Prometheus also supports Recording Rules.

A recording rule evaluates a PromQL expression in advance and stores the result as a new time series.

Example:

- record: instance:node_cpu_usage:rate5m
  expr: |
    100 -
    (
      avg by (instance)
      (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    )

Instead of repeatedly calculating a complex expression, Prometheus can store the calculated result.

## Why Use Recording Rules?

Recording rules are useful when:

Queries are complex
Queries are used frequently
Dashboards use the same calculation repeatedly
Alert rules use expensive queries
We want faster query responses

Flow:

Complex PromQL
      ↓
Recording Rule
      ↓
Stored Time Series
      ↓
Grafana / Alert Rules
## What is Alertmanager?

Alertmanager is a component in the Prometheus monitoring stack that handles alerts generated by Prometheus.

Prometheus is responsible for:

Collect metrics
Evaluate rules
Generate alerts

Alertmanager is responsible for:

Group alerts
Route alerts
Silence alerts
Inhibit alerts
Send notifications
## Prometheus vs Alertmanager
| Prometheus | Alertmanager |
|---|---|
| Collects metrics | Handles alerts |
| Evaluates alert rules | Routes alerts |
| Uses PromQL | Uses routing configuration |
| Generates alerts | Sends notifications |
| Stores time-series data | Groups, silences, and inhibits alerts |

## Alertmanager Workflow
Prometheus
    ↓
Alert generated
    ↓
Alertmanager
    ↓
Routing
    ↓
Grouping
    ↓
Receiver
    ↓
Email / Slack / Other notification
## Alertmanager Route

A route determines where an alert should be sent.

Example:

route:
  receiver: gmail-notifications

This means alerts reaching this route are sent to the configured receiver.

## Alertmanager Receiver

A receiver defines the notification destination.

Example:

receivers:

  - name: gmail-notifications

    email_configs:

      - to: 'prerankumar48@gmail.com'
        send_resolved: true

The receiver is named:

gmail-notifications

and uses email notification.

## Gmail Notification

For our project, Gmail SMTP was used as the email notification mechanism.

The basic flow is:

Prometheus
    ↓
Alert
    ↓
Alertmanager
    ↓
Gmail SMTP
    ↓
Gmail

The SMTP server used was:

smtp.gmail.com:587

TLS was enabled.

## Alertmanager Grouping

Alertmanager can group multiple alerts into a single notification.

Example:

route:
  group_by:
    - namespace

This groups alerts based on namespace.

Instead of receiving separate notifications for every alert immediately, related alerts can be grouped together.

## group_wait

Example:

group_wait: 30s

This specifies how long Alertmanager waits before sending the first notification for a new alert group.

This gives Alertmanager time to collect related alerts.

## group_interval

Example:

group_interval: 5m

This controls how frequently updates to an existing alert group can result in another notification.

## repeat_interval

Example:

repeat_interval: 12h

This controls how often Alertmanager repeats notifications for alerts that remain active.

## Watchdog Alert

The Prometheus monitoring stack commonly includes a Watchdog alert.

The Watchdog alert is used to verify that the alerting pipeline itself is functioning.

In our Alertmanager configuration, Watchdog was routed to a null receiver:

routes:

  - receiver: "null"

    matchers:
      - alertname = "Watchdog"

This prevents the continuously active Watchdog alert from generating unnecessary email notifications.

## Alertmanager Configuration Used in This Project

The Alertmanager configuration contained:

global:
  resolve_timeout: 5m

  smtp_smarthost: 'smtp.gmail.com:587'

  smtp_from: 'prerankumar48@gmail.com'

  smtp_auth_username: 'prerankumar48@gmail.com'

  smtp_auth_password: <GMAIL_APP_PASSWORD>

  smtp_require_tls: true

The actual Gmail App Password must never be committed to Git.

In our repository, the value is represented using a placeholder.

## Secure Credential Handling

Credentials should not be stored directly in Git repositories.

Bad practice:

smtp_auth_password: my-real-password

Better approach:

Kubernetes Secret
       ↓
Environment Variable
       ↓
Grafana / Alertmanager

The actual credential should be stored securely.

## Alert Testing

Creating an alert rule is not enough.

A proper monitoring implementation should test:

Alert condition
      ↓
## Pending
      ↓
## Firing
      ↓
Notification
      ↓
Problem fixed
      ↓
## Resolved
      ↓
Resolved notification

This verifies the complete alerting pipeline.

## Practical Alerting Workflow

Our Kubernetes alerting workflow followed this process:

1\. Identify Kubernetes problem
          ↓
2\. Find appropriate Prometheus metric
          ↓
3\. Write PromQL expression
          ↓
4\. Test expression in Prometheus
          ↓
5\. Create PrometheusRule
          ↓
6\. Configure alert duration
          ↓
7\. Add labels and annotations
          ↓
8\. Prometheus evaluates rule
          ↓
9\. Alert enters Pending
          ↓
10\. Alert becomes Firing
          ↓
11\. Alertmanager receives alert
          ↓
12\. Alertmanager routes notification
          ↓
13\. Gmail receives notification
          ↓
14\. Fix Kubernetes problem
          ↓
15\. Alert becomes Resolved
## Troubleshooting Alert Rules

When an alert does not fire, troubleshoot in the following order.

### Step 1 — Test the PromQL expression

Run the expression directly in Prometheus.

Example:

kube_pod_status_ready{condition="false"} == 1

If there is no result, the alert condition is not currently true.

### Step 2 — Check Prometheus Rules

Verify that Prometheus has loaded the rule.

Useful command:

kubectl get prometheusrules -A
### Step 3 — Check the Prometheus UI

Open the Prometheus alerting/rules pages and verify that the rule exists.

Check whether the alert is:

## Inactive
## Pending
## Firing
### Step 4 — Check Prometheus Operator

Verify that the Prometheus Operator is running:

kubectl get pods -n monitoring

The Prometheus Operator is responsible for managing Prometheus-related Kubernetes resources.

### Step 5 — Check Alertmanager

Verify Alertmanager:

kubectl get pods -n monitoring

The Alertmanager pod should be Running.

### Step 6 — Check Notification Configuration

If the alert fires but no email is received, check:

Alertmanager configuration
SMTP server
SMTP port
Username
App Password
TLS
Receiver
Routing
send_resolved
Common Alerting Problems
| Problem | Possible Cause |
|---|---|
| Alert never fires | PromQL condition is false |
| Alert stays Pending | `for` duration not completed |
| Alert fires but no email | Alertmanager routing/SMTP issue |
| No PrometheusRule | Rule not loaded by Prometheus |
| No data | Incorrect metric or labels |
| Too many emails | Incorrect grouping/repeat interval |
| Watchdog emails | Watchdog not routed to `null` |
| Resolved email missing | `send_resolved` disabled |

## Alerting Best Practices
1\. Use meaningful alert names

Good:

KubernetesPodNotReady
KubernetesPodOOMKilled
KubernetesPodRestarting

Avoid:

Alert1
TestAlert
Problem

for production rules.

2\. Use appropriate thresholds

Avoid alerting on every small temporary change.

Use realistic thresholds and durations.

Example:

for: 5m

instead of immediately firing on a brief spike.

3\. Add useful labels

Example:

labels:
  severity: warning
  category: kubernetes

Labels make routing and filtering easier.

4\. Write useful annotations

The notification should explain:

What happened
Which resource is affected
Why it matters
What should be investigated
5\. Protect credentials

Never commit:

Passwords
App Passwords
API tokens
Access keys
Kubernetes Secrets containing credentials
6\. Test alerts deliberately

A production-quality monitoring setup should be tested by simulating failures.

Examples:

Pod Not Ready
Pod Restart
OOMKilled
CPU Stress
Memory Stress
Image Pull Failure

These practical test scenarios are documented separately in:

07-kubernetes-alert-rules.md
## Practical Alerting Architecture

The final Prometheus-side architecture is:

                Kubernetes
                    │
                    ↓
          kube-state-metrics
                    │
                    ↓
               Prometheus
                    │
              PromQL Rule
                    │
                    ↓
             PrometheusRule
                    │
                    ↓
               Alert Fired
                    │
                    ↓
              Alertmanager
                    │
             ┌──────┴──────┐
             ↓             ↓
          Routing       Grouping
             │
             ↓
          Receiver
             │
             ↓
       Gmail / Slack /
       Teams / PagerDuty
## Key Takeaways

Prometheus alerting allows us to automatically detect infrastructure problems instead of relying only on dashboards.

The important concepts learned in this module are:

PromQL
   ↓
Alert Expression
   ↓
PrometheusRule
   ↓
Alert Evaluation
   ↓
## Pending
   ↓
## Firing
   ↓
Alertmanager
   ↓
Routing
   ↓
Receiver
   ↓
Notification
   ↓
## Resolved

The most important distinction is:

Prometheus detects and generates alerts, while Alertmanager handles, routes, groups, and sends notifications for those alerts.
