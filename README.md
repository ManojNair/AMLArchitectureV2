# Azure Machine Learning Multi-Tenant Architecture

Reference design and rationale for a **single-subscription, multi-workspace Azure Machine Learning (AML) platform** that supports multiple business domains and use cases across dev, test, and prod environments.

This repository is documentation-only. It captures the recommended architecture, the governance model that makes it workable, and how it maps to Microsoft's MLOps v2 guidance.

## Contents

| Document | Purpose |
|---|---|
| [aml_multi_tenant_architecture.md](aml_multi_tenant_architecture.md) | The reference design itself — target architecture, compute model, mixed execution model, naming/metadata standards, batch endpoint reuse, and operating rules. |
| [aml_multi_tenant_architecture_explanation.md](aml_multi_tenant_architecture_explanation.md) | Component-by-component explanation of the architecture, alignment to Microsoft MLOps v2 best practices, and the intentional deviations introduced by the single-subscription constraint. |

## Design at a glance

- **One Azure subscription** (hard constraint)
- **Three AML workspaces**: `aml-dev`, `aml-test`, `aml-prod` — each acts as a multi-tenant execution boundary for its environment
- **Shared AML registry** as the promotion boundary for models, components, and environments
- **Standardised compute classes** instead of per-use-case clusters; one compute instance per data scientist
- **Mixed execution plane**: AML compute clusters, serverless, batch endpoints, managed online endpoints, and AKS for high-demand inferencing
- **Metadata-driven** ownership, cost attribution, and operational routing (since infrastructure is deliberately shared)
- **Batch endpoint reuse** — multiple deployments per endpoint, grouped by domain or processing pattern

## Constraints

- No new subscriptions can be created
- Must stay within AML platform limits (e.g. 1,200 compute targets, 200 batch endpoints per workspace)
- Must support 3 environments and many domains/use cases without one-workspace-per-use-case sprawl

## Audience

ML Platform Engineers, Cloud Architects, Engineering Managers, FinOps, Security, and Operations.

## Status

Draft for internal review.
