**# Module 5 - PromQL Fundamentals**

**## Objective**

In this module, we will learn PromQL (Prometheus Query Language), the query language used by Prometheus to retrieve, filter, aggregate, and analyze time-series metrics.

We will use PromQL to query Kubernetes, node, pod, CPU, memory, and network metrics.

By the end of this module, you will be able to:

\- Understand PromQL
\- Understand Prometheus metrics and labels
\- Execute instant queries
\- Filter metrics using labels
\- Compare metric values
\- Count and aggregate time series
\- Understand Gauge and Counter metrics
\- Use \`rate()\`
\- Use \`increase()\`
\- Query Kubernetes metrics
\- Query CPU and memory metrics
\- Query network metrics
\- Troubleshoot PromQL queries that return no data
\- Build PromQL queries for Grafana dashboards

**---**

**# Prerequisites**

Before starting this module, ensure the following components are available:

\- Kubernetes Cluster
\- Prometheus
\- kube-state-metrics
\- Node Exporter
\- Prometheus UI
\- Grafana

**---**

**# Topics Covered**

\- What is PromQL?
\- Prometheus Time-Series Data
\- Metric Names
\- Labels
\- Instant Queries
\- Label Filtering
\- Comparison Operators
\- Aggregation Functions
\- \`count()\`
\- \`sum()\`
\- Gauge Metrics
\- Counter Metrics
\- \`rate()\`
\- \`increase()\`
\- Kubernetes Pod Metrics
\- CPU Metrics
\- Memory Metrics
\- Network Metrics
\- Practical PromQL Queries
\- Troubleshooting No Data
\- PromQL Best Practices

**---**

**# What is PromQL?**

PromQL stands for:

\> Prometheus Query Language

PromQL is the query language used by Prometheus to select and calculate values from the metrics stored in its time-series database.

PromQL can be used to:

\- Retrieve metrics
\- Filter metrics
\- Compare values
\- Aggregate metrics
\- Calculate rates
\- Calculate increases
\- Analyze Kubernetes resources
\- Build Grafana dashboards
\- Create alert conditions

Example:

\`\`\`promql
up

This retrieves the up metric from Prometheus.

## Prometheus Time-Series Data

Prometheus stores metrics as time-series data.

A time series is identified by:

Metric name
Set of labels
Timestamp
Value

Example:

http_requests_total{method="GET",status="200"}

Here:

Metric name = http_requests_total

Labels:
method = GET
status = 200

The metric value changes over time.

## Metric Names

A PromQL query can start with a metric name.

Example:

up

This retrieves all time series for the up metric.

Another example:

kube_pod_status_phase

This retrieves Kubernetes pod phase information collected by kube-state-metrics.

## Instant Queries

An instant query evaluates a PromQL expression at a single point in time.

In the Prometheus UI, this is useful when we want to see the current value of a metric.

Example:

up

The result shows the current status of Prometheus scrape targets.

Typical values:

1 = Target is UP

0 = Target is DOWN
# Checking Target Status

Query:

up

Example result:

up{instance="localhost:9090"} 1

The value 1 indicates that Prometheus successfully scraped the target.

A value of 0 indicates that the target is currently unavailable.

# Counting Available Targets

We can use the count() aggregation function to count time series.

Query:

count(up)

This returns the number of up time series.

To count only targets that are currently up:

count(up == 1)
# Labels

Labels provide additional information about a metric.

Example:

kube_pod_status_phase{
    namespace="default",
    pod="nginx",
    phase="Running"
}

The metric is:

kube_pod_status_phase

The labels are:

namespace="default"
pod="nginx"
phase="Running"

Labels allow us to filter specific time series.

# Label Filtering

We can filter metrics using label matchers.

Example:

kube_pod_status_phase{phase="Running"}

This returns only the time series where:

phase = Running

Another example:

kube_pod_status_phase{
    namespace="default",
    phase="Running"
}

This filters the results to running pods in the default namespace.

# Common Label Matchers

PromQL supports several label matching operators.

| Operator | Meaning |
|---|---|
| `=` | Equal to |
| `!=` | Not equal to |
| `=~` | Regex match |
| `!~` | Regex does not match |

Examples:

Equal
kube_pod_status_phase{phase="Running"}
Not equal
kube_pod_status_phase{phase!="Pending"}
Regex match
kube_pod_status_phase{namespace=~"default|monitoring"}
Regex does not match
kube_pod_status_phase{namespace!~"kube-system|monitoring"}
# Comparison Operators

PromQL supports comparison operators such as:

\>
<
\>=
<=
==
!=

Example:

up == 1

This selects targets whose up value is 1.

Another example:

kube_pod_status_ready{condition="false"} == 1

This selects pods whose Ready condition is false.

This type of expression is particularly useful when creating alert conditions.

# Aggregation

PromQL provides aggregation functions that allow us to combine multiple time series.

Common aggregation functions include:

# sum()
count()
avg()
min()
max()
# sum()

sum() adds the values of multiple time series.

Example:

sum(up)

This calculates the total value of all up time series.

Another example:

sum(rate(node_network_receive_bytes_total[5m]))

This calculates the total network receive rate across matching interfaces.

# Aggregation by Labels

We can use by() to preserve a specific label while aggregating.

Example:

sum(rate(node_network_receive_bytes_total[5m])) by (instance)

This calculates the network receive rate separately for each instance.

The result is grouped by:

instance
# Gauge Metrics

A Gauge represents a value that can increase or decrease over time.

Examples include:

CPU utilization
Memory usage
Temperature
Number of running pods

A gauge can move in both directions.

Example:

Memory usage:
40%
↓
30%
↓
60%
↓
50%

We normally do not use rate() on a gauge.

# Counter Metrics

A Counter is a metric that normally only increases over time.

Examples include:

Network bytes received
Network bytes transmitted
HTTP requests
Container restarts

A counter can reset when the monitored process restarts.

Example:

100
200
300
400

If the process restarts:

0
100
200

Because counters represent cumulative values, functions such as rate() and increase() are commonly used to analyze them.

# rate()

The rate() function calculates the per-second average increase of a counter over a specified time range.

Example:

rate(node_network_receive_bytes_total[5m])

This calculates the average network receive rate per second over the last 5 minutes.

The range:

[5m]

means Prometheus looks at the previous five minutes of data.

# Why Use rate()?

Suppose a network counter contains:

100 MB
200 MB
300 MB
400 MB

The counter itself represents the total number of bytes received.

If we want to know:

How much traffic is being received per second?

we should calculate its rate.

Therefore:

rate(node_network_receive_bytes_total[5m])

is more useful for a network traffic dashboard than displaying the raw counter.

# increase()

The increase() function calculates the total increase of a counter over a specified time range.

Example:

increase(kube_pod_container_status_restarts_total[1h])

This shows the increase in the container restart counter over the last hour.

Difference between rate() and increase():

| Function | Purpose |
|---|---|
| `rate()` | Per-second average increase |
| `increase()` | Total increase during the selected period |
# Kubernetes Pod Metrics

Kubernetes metrics are exposed through components such as kube-state-metrics.

One useful metric is:

kube_pod_status_phase

It provides information about the phase of Kubernetes pods.

Possible phases include:

Pending
Running
Succeeded
Failed
Unknown
# Running Pods

Query:

kube_pod_status_phase{phase="Running"}

This returns the time series representing running pods.

# Pod Ready Status

Query:

kube_pod_status_ready

To find pods that are not ready:

kube_pod_status_ready{condition="false"} == 1

This query became important later when creating our Kubernetes Pod Not Ready alert.

# Pod Restart Metrics

Container restart information can be queried using:

kube_pod_container_status_restarts_total

Because restart count is a counter, we can use increase() to find restart activity during a time period.

Example:

increase(kube_pod_container_status_restarts_total[1h])
# OOMKilled Metric

To detect containers whose last termination reason was OOMKilled:

kube_pod_container_status_last_terminated_reason{
    reason="OOMKilled"
} == 1

This query was later used as the basis for our Kubernetes OOMKilled alert.

# CPU Metrics

Node Exporter exposes CPU metrics such as:

node_cpu_seconds_total

This is a counter metric.

To calculate CPU utilization, we can calculate the rate of CPU time spent in idle mode.

Example:

100 -
(
  avg by (instance)
  (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) \* 100
)

Conceptually:

CPU Usage = 100% - CPU Idle%
# Memory Metrics

Node Exporter exposes memory metrics such as:

node_memory_MemTotal_bytes

and:

node_memory_MemAvailable_bytes

Memory usage can be calculated using:

(
  node_memory_MemTotal_bytes
  -
  node_memory_MemAvailable_bytes
)
/
node_memory_MemTotal_bytes
\* 100

This gives memory utilization as a percentage.

# Network Metrics

Node Exporter provides network metrics such as:

node_network_receive_bytes_total

This is a Counter.

Therefore, displaying the raw value is usually not useful when we want to visualize current traffic.

We use:

# rate()

to calculate traffic per second.

# Network Traffic Query

The final query used for our Grafana network panel was:

sum(
  rate(
    node_network_receive_bytes_total{device!="lo"}[5m]
  )
) by (instance)

Explanation:

node_network_receive_bytes_total
        ↓
Counter containing cumulative received bytes
        ↓
rate(...[5m])
        ↓
Calculate bytes received per second
        ↓
device!="lo"
        ↓
Exclude loopback interface
        ↓
sum(...)
        ↓
Combine interfaces
        ↓
by (instance)
        ↓
Show result per node
# Why Exclude the `lo` Interface?

Linux has a loopback network interface:

lo

It represents local traffic within the machine.

For a Kubernetes node network dashboard, we generally want to focus on actual network interfaces rather than loopback traffic.

Therefore we used:

{device!="lo"}
# Troubleshooting PromQL: No Data

During the dashboard development, some queries returned:

No data

A No data result does not always mean that Prometheus is broken.

Possible reasons include:

Metric does not exist
Wrong metric name
Wrong label name
Wrong label value
Exporter does not expose the metric
Selected time range has no samples
Query is using the wrong metric type
Metric belongs to a different exporter
# PromQL Troubleshooting Process

When a query returns no data, follow this process.

Step 1 — Check whether the metric exists

Enter only the metric name.

Example:

node_network_receive_bytes_total

If this returns data, the metric exists.

Step 2 — Check the available labels

Inspect the returned series and identify labels such as:

instance
job
device
namespace
pod
container
Step 3 — Add filters gradually

Instead of immediately writing a complex query:

sum(rate(...)) by (...)

start with:

node_network_receive_bytes_total

Then:

node_network_receive_bytes_total{device!="lo"}

Then:

rate(
  node_network_receive_bytes_total{device!="lo"}[5m]
)

Finally:

sum(
  rate(
    node_network_receive_bytes_total{device!="lo"}[5m]
  )
) by (instance)

This makes troubleshooting much easier.

# PromQL in Grafana

PromQL is also used directly inside Grafana panels.

The general workflow is:

Grafana Panel
     ↓
Select Prometheus data source
     ↓
Write PromQL query
     ↓
Run Query
     ↓
Inspect returned time series
     ↓
Choose visualization
     ↓
Configure legend / units / thresholds

PromQL therefore connects the metric collection layer with the visualization layer.

# Instant Query vs Range Query
## Instant Query

An instant query evaluates the expression at one point in time.

Useful for:

Current pod status
Current target status
Current resource values
Alert condition testing

Example:

up
## Range Query

A range query evaluates the expression repeatedly over a time range.

Useful for:

CPU usage over time
Memory usage over time
Network traffic over time
Historical analysis

For example:

rate(node_network_receive_bytes_total[5m])

can be evaluated across a selected Grafana time range to create a time-series visualization.

# Practical PromQL Queries Used in This Project
| Requirement | PromQL |
|---|---|
| Target status | `up` |
| Count UP targets | `count(up == 1)` |
| Pod phases | `kube_pod_status_phase` |
| Running pods | `kube_pod_status_phase{phase="Running"}` |
| Pod Not Ready | `kube_pod_status_ready{condition="false"} == 1` |
| Container restart increase | `increase(kube_pod_container_status_restarts_total[1h])` |
| OOMKilled | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1` |
| Network receive rate | `rate(node_network_receive_bytes_total[5m])` |
| Network traffic by node | `sum(rate(node_network_receive_bytes_total{device!="lo"}[5m])) by (instance)` |
# PromQL Best Practices
1\. Start simple

Instead of immediately writing a complex query:

sum(rate(metric{label="value"}[5m])) by (instance)

start with:

metric

and gradually build the query.

2\. Understand the metric type

Before using a function, determine whether the metric is:

Gauge
Counter

For Counters, functions such as:

# rate()
# increase()

are commonly required.

3\. Use labels carefully

Labels are powerful, but overly broad or incorrect filters can result in:

No data

Always verify the available labels before constructing a complex query.

4\. Use an appropriate time range

Functions such as:

rate(metric[5m])

need enough samples within the selected range.

Very short ranges may sometimes produce insufficient or unstable results.

5\. Aggregate only when necessary

Use:

# sum()
avg()
count()

when you actually need aggregated results.

If you need to identify individual pods or instances, don't aggregate them away.

# Key Takeaways

PromQL is the foundation for querying Prometheus metrics.

The important concepts learned in this module are:

Metric
   ↓
# Labels
   ↓
Filtering
   ↓
Comparison
   ↓
# Aggregation
   ↓
rate() / increase()
   ↓
Kubernetes analysis
   ↓
Grafana visualization
   ↓
Alerting

The most important practical lesson is:

Do not blindly write complex PromQL queries. Start with the metric, verify that data exists, inspect its labels and type, and then gradually build the final query.
