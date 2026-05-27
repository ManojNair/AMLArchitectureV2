# AML Multi-Tenant Architecture – Constraint-Driven Reference Design

## Status
Draft for internal review

## Audience
ML Platform Engineers, Cloud Architects, Engineering Managers, FinOps, Security, Operations

## Purpose
This document captures the recommended **single-subscription, multi-workspace AML platform design** under the current organizational constraints, and explains how the platform should be governed and operated in practice.

This is **not** the unconstrained strategic best-practice design. It is the **best workable design within the current constraints**.

---

# 1. Context and Constraints

The organization has confirmed the following hard constraint:

- **No new subscriptions can be created**
- The AML platform must remain within **one Azure subscription**

At the same time, the platform must support:

- **3 environments**: dev / test / prod
- **Multiple business domains and many use cases**
- Shared AML infrastructure where practical
- A mixed execution model using:
  - AML compute clusters
  - dedicated compute instances
  - serverless
  - batch endpoints
  - managed online endpoints
  - AKS for product-facing or high-demand inferencing

Current key AML platform limits in scope:

- **500 compute targets** (clusters, compute instances, managed online endpoints, etc.)
- **100 batch endpoints**

The design below is intended to stay operable within those limits while avoiding one-workspace-per-use-case sprawl.

---

# 2. Target Architecture

## 2.1 Core Shape

The platform is built as follows:

- **One Azure subscription**
- **Three AML workspaces**
  - `aml-dev`
  - `aml-test`
  - `aml-prod`

Each workspace acts as a **multi-tenant execution boundary** for that environment.

This means:

- All dev use cases run in `aml-dev`
- All test use cases run in `aml-test`
- All prod use cases run in `aml-prod`

There is no dedicated AML workspace per use case.

---

## 2.2 Compute Model

Each environment workspace contains:

- A **small, standardised catalogue of shared compute classes**
- A **dedicated compute instance per data scientist**
- A mix of **execution targets based on workload type**

### Example compute classes
- `cpu-standard`
- `cpu-memory`
- `batch-scoring`
- `gpu-train` / `gpu-infer` (restricted)
- `serverless` (where appropriate)

### Dedicated compute instances
- One compute instance per data scientist per environment
- User-scoped
- Not tied to a particular use case
- Intended for notebooks, local development, VS Code, ad hoc experimentation

### Key principle
Jobs select a **compute class**, not an ad hoc bespoke cluster.

If a workload cannot fit an approved compute class, that should trigger design review rather than automatic creation of a new cluster.

---

## 2.3 Mixed Execution Model

The platform explicitly uses different execution paths for different workload types.

### AML Compute Clusters
Use for:
- training
- AML-native batch workloads
- pipeline-driven scoring
- medium-to-large scheduled jobs

### Serverless
Use for:
- bursty ad hoc experimentation
- AML-native one-off jobs
- scenarios where a managed cluster brings no governance or operational value

### Batch Endpoints
Use for:
- asynchronous batch scoring
- scheduled inferencing jobs
- large data enrichment / parallel scoring workloads

### Managed Online Endpoints
Use for:
- lightweight or medium-scale real-time inference
- AML-hosted APIs where scale and traffic are modest

### AKS
Use for:
- sustained product-facing inferencing
- high-demand online inferencing
- APIs/microservices
- workloads where AML managed endpoints are not the right execution model

### Practical design rule
Do not force AML to host every workload. Use the right execution plane for the right workload.

---

# 3. Naming and Metadata Standards

Use naming for readability only. Use metadata for ownership, cost, observability, and operational routing.

## 3.1 Naming
- Workspaces: `aml-dev`, `aml-test`, `aml-prod`
- Compute clusters: `aml-<tier>-<env>`
  - e.g. `aml-cpu-memory-prod`
- Compute instances: `ci-<alias>-<env>`
- Batch endpoints: `<domain>-batch-<env>` or `<pattern>-batch-<env>`
- Managed online endpoints: `<domain>-rt-<env>` or `<service>-rt-<env>`
- Deployments: `<usecase>-<version>`

## 3.2 Mandatory metadata on jobs / runs / deployments
Every controlled submission path must stamp:

- `domain`
- `usecase`
- `owner`
- `cost_centre`
- `environment`
- `workload_type` (`train`, `batch`, `realtime`, `experimentation`)
- `criticality` (`prod-critical`, `standard`, `low-priority`)

### Why this matters
In a shared workspace, the old trick of inferring ownership from the infrastructure name stops working.

Observability and cost attribution must become **metadata-driven**.

---

# 4. Batch Endpoint Reuse Model

A major objective of this design is to avoid one endpoint per use case.

## 4.1 Principle
A single batch endpoint can support multiple use cases by exposing **multiple deployments** behind the same endpoint.

### Example
```text
Endpoint: pricing-batch-prod

Deployments:
- promo-uplift-v3
- markdown-optimisation-v2
- price-elasticity-v1