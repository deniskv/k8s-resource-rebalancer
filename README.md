# Resource Rebalancer Operator

A Kubernetes operator that continuously reallocates cluster resources from overprovisioned workloads to saturated workloads when horizontal scaling is no longer sufficient.

## Why?

Kubernetes provides several scaling mechanisms:

- HPA (Horizontal Pod Autoscaler)
- VPA (Vertical Pod Autoscaler)
- Cluster Autoscaler
- Karpenter

These tools work well when additional cluster capacity can be provisioned.

However, many environments operate under fixed capacity constraints:

- On-prem Kubernetes clusters
- Budget-constrained cloud environments
- Development and staging clusters
- Air-gapped installations
- Cost-sensitive workloads

In such environments, adding new nodes may not be possible.

The question becomes:

> How can we make better use of resources we already have?

Resource Rebalancer attempts to answer this question.

---

## Concept

Instead of scaling the cluster, the operator continuously identifies:

### Resource Consumers

Workloads that:

- reached HPA limits
- cannot schedule new replicas
- violate SLOs
- experience sustained resource pressure

### Resource Donors

Workloads that:

- consistently consume significantly less than requested resources
- have large unused headroom
- are not under active scaling pressure

The operator gradually transfers resource allocations from donors to consumers.

---

## Example

Before:

Service A

CPU Request: 4000m
Actual Usage: 300m

Service B

CPU Request: 4000m
Actual Usage: 3500m

Service B reaches HPA maxReplicas.

The operator detects:

- Service B is resource constrained
- Service A is heavily overprovisioned

Action:

Service A:
4000m → 3000m

Service B:
4000m → 5000m

The cluster capacity remains unchanged.

Resources are redistributed rather than added.

---

## Goals

- Improve cluster utilization
- Reduce overprovisioning
- Operate under fixed cluster capacity
- Delay or avoid cluster expansion
- Provide recommendations and automated enforcement
- Remain compatible with existing HPA workflows

---

## Non-Goals

The operator is NOT:

- A replacement for HPA
- A replacement for VPA
- A replacement for Cluster Autoscaler
- A scheduler replacement
- A predictive autoscaling system

---

## High-Level Architecture

```text
Prometheus
      |
      v
+----------------+
| Metrics Layer  |
+----------------+
      |
      v
+----------------+
| Observer       |
+----------------+
      |
      v
+----------------+
| Saturation     |
| Detector       |
+----------------+
      |
      v
+----------------+
| Rebalancer     |
+----------------+
      |
      v
+----------------+
| Kubernetes API |
+----------------+
```

## Core Algorithm

### Step 1

Observe:

- Deployments
- StatefulSets
- HPAs
- Pods
- Nodes

### Step 2

Collect metrics:

- CPU usage
- Memory usage
- Requests
- Limits
- HPA state

### Step 3

Detect Saturation

A workload is considered saturated when:

- HPA reached maxReplicas
- resource pressure persists
- SLO degradation observed

### Step 4

Find Donors

Calculate:

headroom = request - P95(actual_usage)

Select workloads with large sustained headroom.

### Step 5

Build Rebalancing Plan

Example:

Need:
+2 CPU

Candidates:

Service A:
headroom 1 CPU

Service B:
headroom 0.5 CPU

Service C:
headroom 1 CPU

Plan:

A: -1 CPU
B: -0.5 CPU
C: -0.5 CPU

Consumer:
+2 CPU

### Step 6

Apply Changes

Mode A:
Recommendation only

Mode B:
Automatic rollout

---

## Safety Mechanisms

### Hysteresis

Prevent oscillation.

### Cooldown Periods

Prevent frequent rebalancing.

### Max Change Per Cycle

Default:
10%

### Priority Awareness

Never steal resources from higher-priority workloads.

### Resource Floors

Never reduce below configured minimums.

---

## Roadmap

### Phase 1

- CPU rebalancing
- Recommendation mode
- HPA saturation detection

### Phase 2

- Automated enforcement
- Memory rebalancing
- Safety policies

### Phase 3

- Budget-aware optimization
- Cluster-wide capacity planning
- Placement optimization

---

## Status

Experimental.

Not recommended for production workloads.

Contributions are welcome.

## Contributing

Issues, design discussions and pull requests are welcome.

Please open a discussion before implementing large architectural changes.