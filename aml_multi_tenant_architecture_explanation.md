# Azure Machine Learning multi-tenant architecture explanation

**Document purpose:** Explain the proposed Azure Machine Learning multi-tenant architecture, how each component fits together, where it aligns to Microsoft MLOps best practices, and where it intentionally deviates because of the stated constraints.

**Architecture scope:** Single Azure subscription, three Azure Machine Learning workspaces for Integration, Staging, and Production, one shared Azure Machine Learning registry, shared data estate, reusable compute classes, shared/domain-pattern batch endpoints, managed online endpoints where appropriate, AKS for high-demand inferencing, centralized observability, and metadata-driven chargeback.

**Primary constraint:** The design is constrained to a **single Azure subscription** and therefore optimizes for operational reuse, cost control, and platform manageability rather than maximum isolation.

**Supporting artifacts used:**

- `AML Multi-Tenant Governance Model.txt`
- `AIPT_Products_Transition_Plan_Updated.xlsx`
- Generated architecture diagram: `azure_machine_learning_architecture_diagram.png`
- Microsoft MLOps v2 classical architecture guidance
- Microsoft Azure Machine Learning registries, workspace, batch endpoint, model monitoring, and cost management guidance

---

## 1. Executive summary

The proposed architecture is a pragmatic Azure Machine Learning platform model for onboarding multiple AIPT use cases into a controlled, reusable, multi-tenant environment.

The architecture follows the broad Microsoft MLOps v2 pattern:

1. Shared enterprise data estate.
2. Platform administration and setup.
3. Model development inner loop.
4. Model registry and CI process.
5. Staging/test and production deployment outer loop.
6. Monitoring, feedback, and retraining actions.

The design aligns well with Microsoft guidance in the following areas:

- It separates development, test/staging, and production using distinct AML workspaces.
- It uses an Azure Machine Learning registry to promote models, components, and environments across workspaces.
- It uses CI/CD gates for build, validation, promotion, and approval.
- It treats monitoring as a first-class part of the lifecycle.
- It explicitly includes automated retraining triggers based on monitoring criteria.
- It uses batch endpoints for asynchronous batch inference.
- It uses managed online endpoints only for low-to-medium demand real-time serving.
- It uses AKS for high-demand or product-facing inferencing where AML managed endpoints would not be the right operating model.
- It uses tags and metadata for chargeback and operational routing.

However, the design is **not a pure Microsoft ideal-state reference architecture**, because the single-subscription and shared-workspace constraints reduce the degree of isolation that Microsoft typically recommends for large enterprise environments. The biggest deviations are:

- Microsoft guidance commonly recommends workspaces as boundaries for access management, cost management, and data isolation, and often suggests a workspace per project for project-level cost reporting. This design instead uses **one workspace per environment**, shared across many use cases.
- Strong business-unit isolation is limited because Azure Machine Learning access control is fundamentally workspace-scoped, not compute-target-scoped.
- Shared compute and shared endpoints introduce noisy-neighbour and endpoint-sprawl risks unless governed tightly.
- Chargeback must be metadata-driven because the infrastructure is deliberately shared.
- AKS introduces a separate operational responsibility model that the platform team must own.

The architecture is therefore best described as:

> A constrained but workable multi-tenant AML operating model that follows MLOps v2 lifecycle principles while relying on governance, metadata, approved submission paths, and operational discipline to compensate for reduced isolation.

---

## 2. Microsoft best-practice alignment baseline

### 2.1 MLOps v2 lifecycle alignment

Microsoft’s MLOps v2 architecture guidance describes machine learning architectures with end-to-end CI/CD pipelines and retraining pipelines. The classical ML architecture specifically includes data estate, administration/setup, model development, registry, staging/test, production deployment, monitoring, and event/action loops.

The proposed diagram maps directly to that lifecycle:

| Microsoft MLOps v2 concept | Architecture component |
|---|---|
| Data estate | Shared data estate |
| Administration and setup | Platform administration and governance |
| Model development / inner loop | AML workspace - Integration |
| Registry | Shared Azure Machine Learning Registry |
| CI/CD promotion | Build, validate, promote, approval gate |
| Staging and test | AML workspace - Staging |
| Production deployment | AML workspace - Production |
| Monitoring | Observability, FinOps and actions |
| Retraining actions | Automated retraining loop based on monitoring criteria |
| Infrastructure actions | Capacity / policy changes loop back to governance |

This is a good architectural fit because the design does not treat Azure Machine Learning as only a batch execution engine. It treats AML as a managed MLOps platform with asset lineage, model promotion, environment separation, deployment governance, monitoring, and retraining feedback loops.

### 2.2 Registry alignment

Azure Machine Learning registries are intended to decouple machine learning assets from individual workspaces and support MLOps across development, testing, and production environments. The proposed architecture uses one shared AML registry to hold reusable models, components, environments, and related ML assets.

This aligns strongly with Microsoft guidance because the registry becomes the promotion boundary between environments rather than relying on copying ad hoc artifacts between workspaces.

### 2.3 Workspace alignment

Azure Machine Learning workspaces provide containers for access management, cost management, data isolation, compute targets, datastores, security settings, logs, metrics, outputs, and lineage metadata.

The design uses three workspaces:

- `aml-integration`
- `aml-staging`
- `aml-production`

This aligns with the common environment-based MLOps pattern, but it deviates from the “workspace per project” option when project-level cost reporting and stronger project isolation are required. The deviation is acceptable only if metadata, tags, RBAC, and operational controls are treated as mandatory platform features rather than optional conventions.

### 2.4 Monitoring and retraining alignment

Microsoft’s model monitoring guidance highlights monitoring for model performance, data drift, prediction drift, data quality, feature attribution drift, and model performance. It also supports event-driven workflows where monitoring events can trigger programmatic actions, including retraining.

The revised architecture explicitly includes:

- Model and data monitoring.
- Automated retraining triggers.
- Schedule-based retraining.
- Data drift triggers.
- Model performance threshold triggers.
- New production data triggers.
- Retrain and validate loop into Staging.
- Model improvement / retraining pipeline loop into Integration.

This is important because automated retraining is not a side note; it is part of the MLOps lifecycle.

### 2.5 Batch endpoint alignment

Batch endpoints are appropriate for asynchronous inference and batch scoring workloads. They provision compute when jobs start and deallocate resources when jobs complete. Microsoft also recommends explicit pipeline components for MLOps practice because they improve reusability and operationalization of complex inference routines.

The proposed architecture uses:

- Shared batch endpoints by domain or processing pattern.
- Multiple deployments behind a batch endpoint.
- AML components and pipelines for reusable scoring and processing logic.
- Shared compute clusters with scale-to-zero where possible.

This aligns with Microsoft’s batch inference model and helps reduce endpoint and compute sprawl.

### 2.6 Cost management and FinOps alignment

Microsoft cost guidance emphasizes planning, estimating, budgeting, monitoring, and reviewing costs across Azure Machine Learning and its associated Azure resources.

The architecture supports this through:

- Mandatory job, endpoint, and deployment tags.
- Cost-centre tagging.
- Environment tagging.
- Workload type tagging.
- Criticality tagging.
- Chargeback/showback dashboards.
- Shared compute classes rather than per-use-case infrastructure.

This is a practical way to support chargeback within a shared infrastructure model, but it only works if tagging is enforced at submission time.

---

## 3. Component-by-component explanation

## 3.1 Single Azure subscription

### Role in the architecture

The entire platform is deployed into a single Azure subscription. This subscription contains:

- Shared data access patterns.
- Three AML workspaces.
- Shared Azure Machine Learning Registry.
- Compute clusters.
- Compute instances.
- Batch endpoints.
- Managed online endpoints.
- Monitoring resources.
- Supporting network, identity, and security controls.

### Why it exists

The single subscription is a stated constraint. It simplifies central governance and avoids the overhead of subscription sprawl, but it reduces isolation and cost-management flexibility.

### Best-practice alignment

This partially aligns with Azure governance practices where a subscription can be used as a management, billing, policy, and access boundary. However, for large-scale enterprise MLOps, separate subscriptions are often preferred for stronger environment isolation, budget separation, and blast-radius reduction.

### Deviation / risk

This is one of the most important deviations from an ideal-state enterprise architecture.

| Risk | Explanation | Mitigation |
|---|---|---|
| Reduced environment isolation | Dev/test/prod share the same subscription boundary. | Use separate AML workspaces, separate resource groups, strict RBAC, Azure Policy, and deployment approvals. |
| Cost attribution is harder | Subscription-level cost reporting cannot automatically separate every use case. | Enforce metadata-driven showback and chargeback. |
| Shared quota pool | Compute quotas are shared across environments unless explicitly planned. | Use quota planning, compute-class limits, and production reservation rules. |
| Larger blast radius | Misconfigured policy or quota exhaustion can affect multiple environments. | Use policy-as-code, resource locks where appropriate, and alerting. |

### Architecture decision

Use the single subscription as the administrative container, but treat AML workspaces, resource groups, identities, network rules, policy, and tags as the effective boundaries.

---

## 3.2 Shared data estate

### Role in the architecture

The shared data estate provides the common curated data used by all environments.

The diagram includes:

- Azure Data Lake Storage Gen2 for curated training data.
- Azure Synapse / SQL / Snowflake as analytical or enterprise data sources.
- Business data sources feeding upstream data products.
- A callout that the same curated training data is reused across all environments.

### Why it exists

The AIPT use cases share the same training data foundation. Reusing curated datasets avoids duplicated data pipelines, inconsistent feature definitions, and unnecessary data copies.

### Best-practice alignment

This aligns with Microsoft’s MLOps architecture pattern where the data estate is separate from the model development and deployment lifecycle. Data engineers own the data estate, while data scientists and ML engineers consume governed data assets.

### Design guidance

The data estate should provide:

- Governed curated datasets.
- Clear data ownership.
- Data-quality checks before ML consumption.
- Versioned data assets or immutable snapshots for reproducibility.
- Consistent training/validation/test dataset definitions.
- Lineage between model versions and training data.

### Important implementation call-outs

1. **Do not allow every use case to build its own private data ingestion pattern.**  
   That would recreate data pipeline sprawl.

2. **Training data reuse must not mean uncontrolled access.**  
   Use datastores, data assets, managed identities, and RBAC.

3. **Production inference data must be captured for monitoring.**  
   For online endpoints, AML can support production data collection. For batch endpoints or external serving, the platform must collect inference inputs, outputs, and ground truth separately.

4. **Automated retraining requires retraining-ready datasets.**  
   The retraining loop only works if fresh production data, reference data, and ground truth are captured and available.

### Best-practice controls

| Control | Recommendation |
|---|---|
| Data access | Use managed identities and least privilege. |
| Data versioning | Register data assets or use immutable data snapshots. |
| Data quality | Run quality checks before training and before retraining. |
| Drift reference | Use training or validation data as baseline for data drift and quality monitoring. |
| Data lineage | Link each registered model to data version, code version, environment, and pipeline run. |

---

## 3.3 Platform administration and governance

### Role in the architecture

This section represents the platform control plane. It includes:

- GitHub / Azure DevOps.
- Bicep / Terraform.
- Workspace and compute provisioning.
- RBAC / Azure Policy.
- Monitors and alerts.
- Approved pipeline templates.

### Why it exists

In a multi-tenant platform, governance cannot be a manual checklist. It must be embedded into provisioning, CI/CD, job submission, and monitoring.

### Best-practice alignment

This aligns with MLOps v2 administration and setup guidance, which includes repository creation, workspace provisioning, compute resource setup, access controls, CI/CD pipelines, and monitors.

### Responsibilities

The platform administration layer should own:

- Workspace creation and baseline configuration.
- Network and private endpoint patterns.
- Managed identity standards.
- RBAC model.
- Compute catalog.
- Endpoint approval process.
- Tagging and metadata policy.
- Pipeline templates.
- Monitoring baselines.
- Cost dashboards.
- Exception review process.

### Required guardrails

The architecture includes the guardrail:

> No ad hoc clusters, no unnecessary endpoint sprawl.

That should be implemented as enforceable operating rules:

| Rule | Why it matters |
|---|---|
| No new compute class without platform review | Prevents compute target sprawl and inconsistent VM choices. |
| No new batch endpoint without reuse assessment | Prevents reaching endpoint limits and operational overload. |
| All jobs submitted through approved templates | Ensures mandatory tags, identities, logging, and compute choices. |
| Premium compute requires approval | Controls GPU/high-memory cost and quota impact. |
| Idle compute instances auto-stop | Reduces waste. |
| Production-critical jobs tagged | Enables prioritisation, alerting, and support routing. |

### Deviation / risk

Because the design uses shared workspaces and shared compute, platform governance becomes a compensating control for the lack of stronger isolation.

If governance is weak, the architecture will fail through predictable patterns:

- Every team asks for custom compute.
- Endpoints grow one per model.
- Tags are missing or inconsistent.
- Production jobs get queued behind exploratory jobs.
- Control-M or monitoring cannot map failures to owners.
- Chargeback becomes disputed.

---

## 3.4 Integration workspace: model development inner loop

### Role in the architecture

The Integration workspace is the primary inner-loop workspace for iterative model development.

It supports:

- Data ingestion.
- Exploratory data analysis.
- Feature engineering.
- Training.
- Evaluation.
- Registration of candidate models.
- Reusable AML components and pipelines.

### Why it exists

This workspace gives data scientists and ML engineers a controlled development environment while keeping development activity separate from staging and production.

### Best-practice alignment

This aligns with the MLOps v2 model development inner loop. The inner loop is where experimentation, model development, evaluation, and candidate model registration occur.

### Components inside the Integration workspace

#### Dedicated compute instances

Compute instances are user-scoped development machines for data scientists.

Use them for:

- Notebook development.
- Debugging.
- Lightweight experimentation.
- Interactive analysis.

Governance rules:

- One compute instance per data scientist per environment only where justified.
- Auto-stop must be enabled.
- No production workloads on compute instances.
- Periodic review of unused compute instances.

Risk:

- With 70-80 data scientists across environments, compute instances can consume a meaningful portion of AML compute target limits and cost.

#### Shared autoscaling compute clusters

Shared clusters run training, feature engineering, and batch workloads.

Recommended standard classes:

| Compute class | Purpose |
|---|---|
| `cpu-standard` | General training and feature engineering. |
| `cpu-memory` | Large-memory models and feature-heavy workloads. |
| `batch-scoring` | Parallel batch inference and scoring workloads. |
| `gpu-train` | Controlled GPU training only by exception. |
| `gpu-infer` | Controlled GPU inference only by exception. |

Best-practice fit:

- Reusable compute classes reduce sprawl.
- Autoscaling helps balance cost and capacity.
- Jobs choose approved compute classes rather than bespoke clusters.

Key risk:

- Azure ML autoscaling adds nodes to the compute cluster, but it does not magically overcommit already allocated nodes. Production-critical jobs need enough available quota and maximum cluster capacity to start on time.

#### Serverless compute

Serverless compute is used for bursty or ad hoc AML-native jobs where owning a cluster adds no value.

Best use cases:

- Short-lived experimentation.
- Low-frequency ad hoc jobs.
- Early-stage development.
- Jobs where capacity management should be abstracted.

Governance point:

- Serverless compute should still enforce tags, identity, cost centre, and workload type.

#### Reusable AML components and pipelines

AML components standardize reusable steps, such as:

- Data extraction.
- Feature transformation.
- Model training.
- Evaluation.
- Batch scoring.
- Data validation.
- Drift analysis.
- Model registration.

Best-practice fit:

- Components improve reuse.
- Components support pipeline standardisation.
- Components help create repeatable, auditable MLOps workflows.
- Components are reusable across workspaces when shared through the registry.

Important distinction:

> Components help reuse logic. Deployments help reuse endpoints. They are not the same thing.

### Deviation / risk

The Integration workspace is shared by many use cases. That improves reuse and cost efficiency but reduces team-level isolation.

Mitigations:

- Use Microsoft Entra groups for workspace access.
- Use approved templates.
- Use job-level tags.
- Use pipeline-level identities.
- Use compute-class access through controlled submission paths.
- Use review gates for premium compute.

---

## 3.5 Shared Azure Machine Learning Registry and CI

### Role in the architecture

The shared AML registry is the cross-workspace asset catalog and promotion mechanism.

It stores and shares:

- Models.
- Components.
- Environments.
- Datasets or data assets where appropriate.

### Why it exists

The registry allows a candidate model developed in Integration to be promoted and reused in Staging and Production without treating each workspace as an isolated island.

### Best-practice alignment

This is one of the strongest best-practice alignments in the architecture. Azure Machine Learning registries are explicitly designed to support cross-workspace MLOps and asset sharing across development, test, and production environments.

### CI/CD promotion flow

The diagram shows:

1. Build.
2. Validate.
3. Promote.
4. Approval gate.

Recommended flow:

| Stage | Description |
|---|---|
| Build | Package code, components, environments, and pipeline definitions. |
| Validate | Run unit tests, data-contract tests, component tests, security checks, and reproducibility checks. |
| Register | Register candidate model and associated assets. |
| Promote | Promote model/component/environment into the registry. |
| Approve | Human-in-the-loop approval for production promotion. |
| Deploy | Deploy to staging, test, then production. |

### Required metadata on registry assets

Each promoted model should carry or link to:

- Model name and version.
- Source training job.
- Code commit.
- Environment version.
- Component version.
- Data asset version.
- Training metrics.
- Evaluation metrics.
- Responsible AI checks.
- Owner.
- Use case.
- Risk/criticality.
- Approval status.

### Deviation / risk

The registry does not replace governance. It enables promotion, but it does not by itself decide whether a model is ready for production.

Required controls:

- Approval gate.
- Model risk review where required.
- Responsible AI review.
- Security review for production endpoints.
- Data lineage validation.
- Rollback plan.

---

## 3.6 Staging workspace: validation, retraining, and test deployments

### Role in the architecture

The Staging workspace is the pre-production environment.

It handles:

- Validation.
- Retraining.
- Data quality checks.
- Responsible AI checks.
- Unit tests.
- Test deployments.
- Performance validation.
- Endpoint validation.

### Why it exists

Staging is the environment where a model candidate proves that it is fit for production before production deployment.

### Best-practice alignment

This aligns directly with the MLOps v2 staging and test phase, where model candidates are tested, validated, and potentially retrained using production-like data and deployment patterns.

### Staging activities

#### Validation and retraining

Staging is where automated retraining candidates should be evaluated before production promotion.

Examples:

- Scheduled retraining produces a candidate model.
- Model monitoring detects drift and triggers retraining.
- Production performance degradation triggers retraining.
- New production data becomes available and triggers a retraining pipeline.

The retrained model should not go straight into production. It should pass staging gates first.

#### Data quality checks

Data quality checks should validate:

- Schema.
- Null rates.
- Outliers.
- Range violations.
- Feature distribution shifts.
- Unexpected categorical values.
- Training-serving skew.
- Missing ground truth availability.

#### Responsible AI and unit tests

Responsible AI and quality checks should include:

- Bias/fairness checks where applicable.
- Explainability review where required.
- Performance by segment.
- Unit tests for scoring code.
- Integration tests for pipeline components.
- Security checks for dependencies and container images.

#### Test deployments

Staging should host test versions of:

- Shared batch endpoint deployments.
- Managed online endpoint deployments.
- AKS deployment candidates where appropriate.

### Deviation / risk

If Staging shares too much compute and too many endpoint patterns with Production, noisy-neighbour risk and accidental production impact increase.

Mitigations:

- Separate staging compute clusters from production compute clusters.
- Separate staging endpoints from production endpoints.
- Separate identities.
- Separate approval gates.
- Use production-like test data without granting unnecessary production access.

---

## 3.7 Production workspace: controlled deployment and serving

### Role in the architecture

The Production workspace hosts approved models and production-serving patterns.

It includes:

- Shared batch endpoints by domain or processing pattern.
- Multiple deployments behind shared endpoints.
- Managed online endpoints for low-to-medium real-time demand.
- AKS inference services for high-demand product-facing APIs.
- Control-M or orchestrators for scheduled batch jobs.
- Shared autoscaling compute clusters for batch workloads.

### Why it exists

Production is the controlled environment where approved models generate business outcomes.

### Best-practice alignment

This aligns with MLOps v2 production deployment guidance, which supports batch managed endpoints for batch scenarios and online or Kubernetes-based options for near-real-time scenarios.

### Production deployment options

#### Shared batch endpoints by domain / pattern

The diagram shows batch endpoints grouped by domain or pattern, with deployments such as:

- Range Optimiser.
- Promo Manager.
- Customer Models.

This is deliberately different from creating one endpoint per model or per use case.

Recommended endpoint design:

| Endpoint pattern | When to use |
|---|---|
| Domain endpoint | Multiple related models in the same business domain. |
| Processing-pattern endpoint | Multiple models that share similar input/output and execution patterns. |
| Dedicated endpoint | Only when isolation, SLA, security, or operational needs justify it. |

Why this is best within the constraint:

- Reduces endpoint sprawl.
- Centralizes operational monitoring.
- Supports shared orchestration.
- Allows multiple deployments behind a single endpoint.
- Supports model versioning without endpoint-per-version proliferation.

Risk:

- Over-consolidation can make one endpoint too complex or operationally noisy.

Mitigation:

- Use a clear endpoint taxonomy.
- Separate criticality tiers.
- Use deployment-level metadata.
- Use clear ownership tags.
- Review new endpoint requests.

#### Control-M / orchestrators

Control-M or another enterprise scheduler triggers batch jobs.

It should pass:

- Use case.
- Model version or deployment name.
- Input data reference.
- Output destination.
- Business date.
- Correlation ID.
- Cost centre.
- Criticality.
- SLA window.

Key design point:

> Control-M should track job identity and deployment identity, not just endpoint names.

This is essential because shared endpoints can host multiple deployments and use cases.

#### Managed online endpoints

Managed online endpoints are suitable for:

- Low-to-medium demand real-time inference.
- Simple real-time use cases.
- Cases where AML-managed deployment, autoscaling, and blue-green/canary deployment features are valuable.

Best-practice fit:

- Azure Machine Learning supports managed operational capabilities for endpoints.
- A single endpoint can support multiple deployments for rollout patterns.

Risk:

- For high-demand product-facing APIs, managed online endpoints can become expensive or operationally limiting compared with a dedicated application/API serving layer.

Mitigation:

- Use managed online endpoints deliberately.
- Define thresholds for when to move to AKS.
- Monitor cost, latency, throughput, and scale behavior.

#### AKS inference services

AKS is used for sustained or high-demand product-facing inferencing APIs.

Use AKS when:

- You need fine-grained scaling control.
- You need product-grade API routing.
- You need custom ingress/API gateway patterns.
- You need advanced traffic management.
- You need application-specific retries, circuit breakers, or custom observability.
- You need to integrate with broader application platform patterns.

Important caveat:

> AKS is not “just another endpoint.” It is a different operating model.

Responsibilities shift to the platform/application team:

- Container build and patching.
- API routing.
- Ingress/TLS.
- Autoscaling.
- Rollout strategy.
- Observability.
- Security.
- Network policy.
- Reliability engineering.
- Incident response.

### Deviation / risk

Using AKS can be a best-practice choice for high-demand workloads, but only if the organization accepts the operational responsibility. If the platform team is not ready to operate AKS as a production application platform, AKS becomes a risk rather than a solution.

---

## 3.8 Observability, FinOps, and actions

### Role in the architecture

This section centralizes operational visibility and actions.

It includes:

- Azure Monitor.
- Log Analytics.
- Application Insights.
- Model and data monitoring.
- Infrastructure monitoring.
- Chargeback/showback dashboards.
- Automated retraining triggers.
- Metadata-driven chargeback tags.

### Why it exists

A shared multi-tenant platform must be observable by metadata, not by infrastructure ownership. In the old model, one workspace per use case made ownership obvious. In the new model, ownership must be inferred from job metadata, deployment IDs, tags, and run IDs.

### Best-practice alignment

This aligns strongly with Microsoft guidance around model monitoring, cost management, and MLOps lifecycle feedback loops.

### Observability dimensions

#### Model and data monitoring

Monitor:

- Data drift.
- Prediction drift.
- Data quality.
- Feature attribution drift.
- Model performance.
- Ground truth availability.
- Business KPI degradation.
- Segment-level performance.

Monitoring outputs should support:

- Alerts.
- Dashboards.
- Investigation.
- Retraining triggers.
- Model retirement decisions.

#### Infrastructure monitoring

Monitor:

- Endpoint latency.
- Endpoint failures.
- Batch job duration.
- Queue time.
- Compute capacity.
- Cluster autoscaling behavior.
- Node failures.
- Quota pressure.
- AKS pod health.
- API errors.
- Dependency failures.
- Network issues.

#### FinOps and chargeback

Chargeback/showback should be based on:

- Job tags.
- Endpoint tags.
- Deployment tags.
- Run IDs.
- Compute usage.
- Duration.
- VM SKU.
- Environment.
- Cost centre.
- Owner.
- Criticality.

Mandatory tags:

- `domain`
- `usecase`
- `owner`
- `cost_centre`
- `environment`
- `workload_type`
- `criticality`

### Key principle

> In a shared platform, tags are not optional labels. They are the operating model.

If tags are missing, the platform cannot reliably support ownership, chargeback, support routing, prioritisation, or SLA reporting.

---

## 3.9 Automated retraining loop

### Role in the architecture

The automated retraining loop connects production monitoring back into model improvement and validation.

It includes:

- Monitoring signals.
- Automated retraining triggers.
- Retraining pipeline.
- Validation in Staging.
- Optional model improvement in Integration.
- Gated promotion back to Production.

### Why it exists

Models become stale because production data changes, user behavior changes, external conditions change, or business rules evolve. The architecture needs a feedback loop that converts monitoring signals into controlled retraining actions.

### Trigger types

The diagram includes four retraining triggers:

| Trigger | Description |
|---|---|
| Schedule-based retraining | Retraining runs on a fixed cadence, such as weekly, monthly, or quarterly. |
| Data drift | Production feature distributions diverge from the training or reference baseline. |
| Model performance thresholds | Accuracy, error, ranking quality, or business KPI drops below threshold. |
| New production data | New labelled or validated production data becomes available. |

### Recommended retraining flow

1. Production model monitoring detects a condition.
2. Event or schedule triggers retraining workflow.
3. Retraining job uses approved pipeline template.
4. Job stamps mandatory metadata.
5. Candidate model is trained.
6. Candidate model is evaluated.
7. Candidate model is registered.
8. Candidate model is validated in Staging.
9. Data quality, unit, Responsible AI, and performance checks run.
10. Approval gate decides whether to promote.
11. Model is promoted through the registry.
12. Production deployment is updated using controlled release strategy.
13. Monitoring continues.

### Best-practice alignment

This directly aligns with the MLOps v2 idea that monitoring events and criteria can cause automated actions, including scheduled retraining or retraining based on model/data issues.

### Important design call-out

Automated retraining should not mean automated production deployment.

The recommended pattern is:

> Automated retraining, automated validation, human or policy-based approval, controlled production deployment.

For low-risk models, approval can be policy-based. For high-impact models, human approval should remain.

---

## 3.10 Infrastructure feedback loop

### Role in the architecture

Infrastructure monitoring can trigger capacity or policy changes back into platform governance.

Examples:

- Endpoint latency exceeds threshold.
- Batch jobs miss SLA windows.
- Compute queue time is too high.
- Quota is exhausted.
- Cluster scale-out is insufficient.
- AKS service requires more replicas or nodes.
- Network or dependency errors increase.
- Compute costs exceed forecast.

### Best-practice alignment

This aligns with MLOps v2 infrastructure monitoring and action loops. It also aligns with Azure cost and operational excellence practices.

### Recommended actions

| Signal | Action |
|---|---|
| Batch job queue time increasing | Increase max nodes, add workload-specific compute class, or reschedule workloads. |
| Production-critical job delayed by lower-priority jobs | Separate compute class or enforce priority submission. |
| Endpoint latency too high | Scale endpoint, optimize model, move workload to AKS, or split endpoint. |
| Compute cost spike | Review tags, workload schedule, SKU choice, and low-priority VM eligibility. |
| Endpoint count increasing | Enforce endpoint review and consolidate deployments. |
| GPU usage increasing | Require approval and quota planning. |

---

## 4. Use case fit

The workbook identifies AIPT use cases across product groups such as:

- Range Optimiser / Store Specific Program.
- Promo App / Promo Manager.
- Smarter Clearance.
- Flash Sales.
- Customer Models / segmentation.

The architecture supports these through a common onboarding pattern:

1. Classify workload type:
   - Training.
   - Batch scoring.
   - Real-time scoring.
   - Experimentation.
   - High-demand API inference.

2. Map to compute class:
   - `cpu-standard`
   - `cpu-memory`
   - `batch-scoring`
   - `serverless`
   - `managed online endpoint`
   - `AKS`

3. Map to endpoint pattern:
   - Domain batch endpoint.
   - Processing-pattern batch endpoint.
   - Managed online endpoint.
   - AKS inference API.

4. Stamp required metadata:
   - Domain.
   - Use case.
   - Owner.
   - Cost centre.
   - Environment.
   - Workload type.
   - Criticality.

5. Promote assets through registry:
   - Integration to Staging to Production.

6. Monitor and charge back:
   - Run ID.
   - Deployment ID.
   - Endpoint.
   - Compute usage.
   - SLA.
   - Cost centre.

### Example mapping

| Use case group | Likely serving pattern | Notes |
|---|---|---|
| Range Optimiser / Store Specific Program | Batch endpoint, possibly AKS for product-facing APIs | Large scheduled workloads should use shared batch compute; product APIs may belong on AKS. |
| Promo App / Promo Manager | Batch endpoint and/or managed online endpoint | Use shared domain endpoint where possible. |
| Smarter Clearance | Batch endpoint or real-time endpoint depending on business process | Criticality and SLA determine compute separation. |
| Flash Sales | Potentially time-sensitive batch or real-time | Needs SLA and peak-demand review. |
| Customer Models / segmentation | Batch endpoint | Good fit for scheduled segmentation scoring. |

---

## 5. Best-practice alignment matrix

| Architecture decision | Alignment with Microsoft best practice | Assessment |
|---|---|---|
| Separate Integration, Staging, and Production AML workspaces | Aligns with environment-based MLOps separation. | Strong alignment. |
| Shared AML registry | Aligns with cross-workspace MLOps and asset reuse. | Strong alignment. |
| CI/CD promotion with approval gate | Aligns with MLOps v2 outer-loop deployment. | Strong alignment. |
| Shared curated data estate | Aligns with data estate separation in MLOps v2. | Strong alignment, provided access is governed. |
| Reusable AML components and pipelines | Aligns with repeatable and maintainable MLOps. | Strong alignment. |
| Batch endpoints for asynchronous scoring | Aligns with AML batch endpoint purpose. | Strong alignment. |
| Multiple deployments behind shared endpoints | Aligns with endpoint reuse and deployment lifecycle management. | Strong alignment when governed. |
| Managed online endpoints for low/medium demand | Aligns with managed inference capabilities. | Good alignment. |
| AKS for high-demand product APIs | Aligns when advanced control and scale are needed. | Good alignment but higher operational burden. |
| Model monitoring and retraining triggers | Aligns with MLOps v2 monitoring/actions and AML model monitoring. | Strong alignment. |
| Metadata-driven chargeback | Aligns with cost management needs in shared platforms. | Necessary due to constraints. |
| One workspace per environment, shared by many use cases | Deviates from project-level workspace isolation guidance. | Acceptable only with strong governance. |
| Single subscription for all environments | Less ideal for enterprise isolation and budget separation. | Constraint-driven deviation. |
| Shared compute clusters | Cost-efficient but increases noisy-neighbour risk. | Acceptable only with compute governance. |
| Shared batch endpoints | Reduces sprawl but can concentrate operational risk. | Acceptable with endpoint taxonomy and ownership metadata. |

---

## 6. Deviation and issue register

### 6.1 Single subscription

**Issue:** A single subscription is not the strongest boundary for large enterprise MLOps environments.

**Why this matters:** Subscriptions are often used for billing, quota, policy, and blast-radius separation. Combining Integration, Staging, and Production in one subscription creates shared quota and governance dependencies.

**Mitigation:**

- Separate resource groups per environment.
- Separate AML workspaces per environment.
- Strict RBAC.
- Azure Policy.
- Budget alerts.
- Per-environment tags.
- Quota planning.
- Production compute reservation rules.

**Residual risk:** Medium.

---

### 6.2 Workspace-per-environment instead of workspace-per-project

**Issue:** Microsoft workspace guidance notes that limiting a workspace to one project helps with project-level cost reporting and scoped configurations. This architecture uses one workspace per environment across many use cases.

**Why this matters:** Ownership, access control, troubleshooting, and cost attribution become metadata-driven rather than resource-driven.

**Mitigation:**

- Mandatory tags.
- Approved submission paths.
- Job/run metadata.
- Deployment IDs.
- Cost dashboards.
- Use-case onboarding reviews.
- Periodic workspace hygiene reviews.

**Residual risk:** Medium to high if tagging is not enforced.

---

### 6.3 No compute-target-level RBAC

**Issue:** Access control is workspace-scoped. The platform should not assume native “Team A can submit to cluster X but not cluster Y” enforcement inside a shared workspace.

**Why this matters:** Teams could accidentally or intentionally use compute classes not intended for them.

**Mitigation:**

- Controlled submission via approved pipeline templates.
- Separate service principals or managed identities for specific workload classes.
- Premium compute approval gates.
- Logging and alerting for unauthorized compute usage.
- Separate workspace if hard isolation becomes mandatory.

**Residual risk:** Medium.

---

### 6.4 Noisy-neighbour risk

**Issue:** Shared compute clusters can cause contention.

**Why this matters:** One team’s large workload can delay another team’s SLA-critical job.

**Mitigation:**

- Separate compute classes by workload purpose.
- Production-critical tagging.
- SLA-based scheduling.
- Soft quotas by domain.
- Separate production batch compute.
- Review of queue time and job duration.
- Dedicated compute only when justified.

**Residual risk:** Medium.

---

### 6.5 Endpoint sprawl

**Issue:** Creating one endpoint per use case recreates the scaling problem in a different form.

**Why this matters:** Endpoint count, management overhead, monitoring complexity, and operational support all increase.

**Mitigation:**

- Domain/pattern endpoint taxonomy.
- Multiple deployments behind shared endpoints.
- New endpoint review process.
- Endpoint owner metadata.
- Retire unused deployments.
- Avoid endpoint-per-version patterns.

**Residual risk:** Medium.

---

### 6.6 Compute target sprawl

**Issue:** Compute instances, clusters, managed online endpoints, and other compute targets can accumulate.

**Why this matters:** The platform can hit service limits or become operationally unmanageable.

**Mitigation:**

- Small approved compute catalog.
- Auto-stop compute instances.
- Periodic cleanup.
- Retirement process for unused endpoints.
- Platform approval for new compute classes.
- Quota and limit dashboards.

**Residual risk:** Medium.

---

### 6.7 AKS operational burden

**Issue:** AKS is appropriate for high-demand inferencing, but it shifts operational responsibility to the platform/application team.

**Why this matters:** AKS requires ownership of scaling, ingress, TLS, monitoring, patching, security, and incident response.

**Mitigation:**

- Use AKS only where demand or product-facing requirements justify it.
- Define AKS platform standards.
- Use GitOps or CI/CD.
- Use managed identities.
- Use network policies.
- Use centralized logging and tracing.
- Define SLOs.
- Use API gateway/ingress patterns.
- Ensure platform team owns runtime operations.

**Residual risk:** Medium to high without mature AKS operations.

---

### 6.8 Metadata dependency for chargeback

**Issue:** Chargeback depends on complete and accurate metadata.

**Why this matters:** Shared infrastructure means cost cannot always be attributed by resource name.

**Mitigation:**

- Reject jobs without mandatory tags.
- Stamp tags automatically through templates.
- Validate tags in CI/CD.
- Include tags on jobs, endpoints, deployments, and data assets.
- Use dashboards that group by tags.
- Perform monthly tag compliance reviews.

**Residual risk:** High if not automated.

---

### 6.9 Automated retraining governance

**Issue:** Automated retraining can introduce risk if retrained models are deployed without validation.

**Why this matters:** A retrained model can perform worse, introduce bias, or break downstream consumers.

**Mitigation:**

- Automated retraining does not equal automatic production deployment.
- Require Staging validation.
- Require performance gates.
- Require rollback plan.
- Use approval gates based on model criticality.
- Track lineage for retrained models.
- Store evaluation artifacts.

**Residual risk:** Low to medium with gates; high without gates.

---

## 7. Required operating model

### 7.1 Onboarding checklist for a new use case

Every new use case should answer:

1. What is the business domain?
2. Who owns the model?
3. What is the cost centre?
4. Is the workload training, batch, real-time, or experimentation?
5. What is the criticality?
6. What SLA applies?
7. Does it need batch endpoint, managed online endpoint, or AKS?
8. Can it reuse an existing endpoint?
9. Can it reuse existing components?
10. What compute class does it require?
11. Does it need premium compute?
12. What data assets are required?
13. What monitoring signals are required?
14. What retraining trigger applies?
15. What approval gate is required?
16. What rollback process applies?

### 7.2 Mandatory metadata contract

All AML jobs, endpoints, deployments, and production pipelines should include:

```yaml
tags:
  domain: "<business-domain>"
  usecase: "<use-case-name>"
  owner: "<team-or-service-owner>"
  cost_centre: "<cost-centre>"
  environment: "integration|staging|production"
  workload_type: "train|batch|realtime|experimentation"
  criticality: "prod-critical|standard|low-priority"
```

Recommended additional tags:

```yaml
  model_name: "<model-name>"
  model_version: "<model-version>"
  data_asset: "<data-asset-name>"
  data_version: "<data-version>"
  git_commit: "<commit-sha>"
  pipeline_name: "<pipeline-name>"
  orchestrator: "control-m|azure-devops|github-actions|manual"
  sla_window: "<business-sla-window>"
  pii: "yes|no"
  retraining_policy: "scheduled|drift|manual|none"
```

### 7.3 Compute class policy

| Class | Allowed use | Approval required |
|---|---|---|
| `cpu-standard` | General training, feature engineering, small-to-medium batch | No |
| `cpu-memory` | Memory-heavy models and feature engineering | Yes for large SKU |
| `batch-scoring` | Parallel batch scoring | No, if within quota |
| `serverless` | Ad hoc and bursty AML-native workloads | No, but tags mandatory |
| `gpu-train` | GPU training | Yes |
| `gpu-infer` | GPU inference | Yes |
| `managed-online` | Low/medium real-time | Yes for production |
| `aks-infer` | High-demand product APIs | Architecture review required |

### 7.4 Endpoint governance policy

A new batch endpoint should be approved only when:

- Existing domain/pattern endpoint is unsuitable.
- SLA or security isolation requires it.
- Input/output contract differs significantly.
- Operational ownership is clear.
- Monitoring and chargeback tags are defined.
- Retirement policy is documented.

Otherwise, add a new deployment behind an existing endpoint.

---

## 8. Recommended implementation backlog

### Phase 1: Foundation

- Create Integration, Staging, and Production AML workspaces.
- Create shared AML registry.
- Establish resource groups and naming standards.
- Implement RBAC groups.
- Implement baseline Azure Policy.
- Implement shared data access pattern.
- Define compute catalog.
- Define endpoint taxonomy.
- Create mandatory tagging schema.

### Phase 2: MLOps templates

- Create approved training pipeline template.
- Create approved batch scoring pipeline template.
- Create model registration template.
- Create registry promotion workflow.
- Create staging validation workflow.
- Create production deployment workflow.
- Add tag validation checks.
- Add model lineage capture.

### Phase 3: Observability and FinOps

- Enable Log Analytics and Application Insights.
- Build AML job dashboard.
- Build endpoint dashboard.
- Build compute utilization dashboard.
- Build queue-time dashboard.
- Build chargeback/showback dashboard.
- Add tag-compliance dashboard.
- Add alert rules for SLA and cost anomalies.

### Phase 4: Automated retraining

- Define retraining policies by use case.
- Enable model/data monitoring.
- Define thresholds.
- Capture production inference data.
- Integrate Event Grid or scheduler-based triggers.
- Implement retraining pipeline.
- Validate retrained models in Staging.
- Implement approval and production release gates.

### Phase 5: Production hardening

- Define AKS inference standards.
- Define managed online endpoint standards.
- Define rollback patterns.
- Define incident response.
- Define business continuity rules.
- Define quota and capacity runbooks.
- Perform architecture review for high-criticality use cases.

---

## 9. Final assessment

The architecture is directionally sound and aligns well with Microsoft’s MLOps lifecycle guidance, especially around:

- Environment-separated workspaces.
- Registry-based promotion.
- CI/CD and approval gates.
- Reusable components and pipelines.
- Batch endpoints for asynchronous scoring.
- Model monitoring and retraining actions.
- Cost and operational visibility.

The main concern is not whether Azure Machine Learning can support the pattern. It can. The concern is whether the operating model will be strong enough to compensate for the deliberate multi-tenant constraints.

The design should be approved only with the following non-negotiables:

1. Mandatory metadata on every job, endpoint, deployment, and pipeline.
2. Approved submission paths for production workloads.
3. Compute catalog with strict review for new classes.
4. Endpoint taxonomy with review before new endpoint creation.
5. Model registry promotion and approval gates.
6. Model/data monitoring from day one.
7. Automated retraining routed through Staging validation.
8. Chargeback/showback based on metadata, not resource ownership.
9. Separate AKS operating model for high-demand APIs.
10. Regular governance review for compute, endpoints, tags, cost, and SLA performance.

Within the single-subscription constraint, this is a reasonable and defensible architecture. Without governance automation and mandatory metadata enforcement, it will drift into the same failure modes it is trying to avoid: compute sprawl, endpoint sprawl, weak observability, and unclear ownership.

---

## 10. Source references

Microsoft sources:

- Microsoft Azure Architecture Center — Machine learning operations: https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/machine-learning-operations-v2
- Microsoft Learn — Machine Learning registries for MLOps: https://learn.microsoft.com/en-us/azure/machine-learning/concept-machine-learning-registries-mlops?view=azureml-api-2
- Microsoft Learn — Share models, components, and environments across workspaces with registries: https://learn.microsoft.com/en-us/azure/machine-learning/how-to-share-models-pipelines-across-workspaces-with-registries?view=azureml-api-2
- Microsoft Learn — What is an Azure Machine Learning workspace?: https://learn.microsoft.com/en-us/azure/machine-learning/concept-workspace?view=azureml-api-2
- Microsoft Learn — Azure Machine Learning model monitoring: https://learn.microsoft.com/en-us/azure/machine-learning/concept-model-monitoring?view=azureml-api-2
- Microsoft Learn — Batch endpoints: https://learn.microsoft.com/en-us/azure/machine-learning/concept-endpoints-batch?view=azureml-api-2
- Microsoft Learn — Plan to manage costs for Azure Machine Learning: https://learn.microsoft.com/en-us/azure/machine-learning/concept-plan-manage-cost?view=azureml-api-2
- Microsoft Learn — MLOps and GenAIOps for AI workloads on Azure: https://learn.microsoft.com/en-us/azure/well-architected/ai/mlops-genaiops
- Microsoft Learn — MLOps best practices in Azure Kubernetes Service: https://learn.microsoft.com/en-us/azure/aks/best-practices-ml-ops

Uploaded/user-provided sources:

- `AML Multi-Tenant Governance Model.txt`
- `AIPT_Products_Transition_Plan_Updated.xlsx`

