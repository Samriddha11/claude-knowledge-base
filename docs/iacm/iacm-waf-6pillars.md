---
title: IaCM Well-Architected Framework — 6 Pillars Guide
subtitle: Six Pillars for Production-Grade Infrastructure as Code Management
customer: Harness
date: May 2026
author: Harness IaCM Centre of Excellence
category: iacm
tags:
  - iacm
  - terraform
  - well-architected
  - security
  - finops
---

# Harness IaCM Well-Architected Framework

**Six Pillars for Production-Grade Infrastructure as Code Management**

> *A strategic assessment framework for enterprise infrastructure teams evaluating or scaling Harness IaCM adoption.*

---

| | |
|---|---|
| **Author** | Harness IaCM Centre of Excellence |
| **Version** | 2.0 |
| **Date** | May 2026 |
| **Classification** | Internal |

---

## About This Framework

Infrastructure as Code failed most enterprises not because the tools were wrong, but because the **governance wrapped around them was nonexistent**. This framework answers one question: what does it take to run IaC at enterprise scale — consistently, securely, and without surprising the finance team at month-end?

Each pillar contains design principles, real-world consequence examples, maturity indicators, and capability tables. Use the self-assessment at the end of each pillar to score your current state (1–5) and identify where to focus.

---

## The Six Pillars at a Glance

| # | Pillar | Core Question |
|---|--------|---------------|
| 1 | Pipeline Design & Lifecycle | Are your IaC pipelines consistent, composable, and complete? |
| 2 | Security, Policy & Compliance | Is every infrastructure change scanned, governed, and audited? |
| 3 | State Management & Reliability | Is your Terraform state protected, versioned, and recoverable? |
| 4 | Module Registry & Workspace Governance | Are infrastructure building blocks reusable and consistently deployed? |
| 5 | Cost & FinOps Integration | Do engineers see the cost impact of changes before they apply them? |
| 6 | Developer Experience & Enablement | Can teams self-serve IaCM without platform engineering intervention? |

---

## Pillar 1 — Pipeline Design & Lifecycle

> *"A well-designed IaCM pipeline is the single source of truth for how infrastructure changes flow from code to cloud."*

### Design Principles

1. **Every lifecycle action has a pipeline.** Plan, apply, destroy, and drift detection are first-class pipeline citizens — not ad-hoc terminal operations.

2. **One canonical pattern per environment tier.** Dev, staging, and production share the same pipeline template with environment-scoped variable overrides — not separate YAML files per environment.

3. **Templates at the highest practical scope.** Security scans, approval gates, and notifications live at account- or org-level templates. Changes propagate to all pipelines automatically.

4. **Pipelines are observable.** Every run produces structured logs, cost delta, scan results, and drift status — surfaced in a centralised dashboard, not buried in execution logs.

### Pipeline Lifecycle

```
CLONE → INIT → PLAN → [ SCAN ZONE ] → GATE → ACT → VERIFY

Scan Zone (sequential):
  ① Checkov / tfsec  →  ② Wiz IaC  →  ③ OPA Policies  →  ④ Infracost

Gate:   IaCMApproval (plan diff + cost delta visible to approver)
Act:    apply | destroy | dry-run  ← controlled by TF_Action stage variable
Verify: detect-drift + output export to CD pipeline
```

### Real-World Consequence

> A financial services firm with 40 engineers running `terraform apply` from laptops discovered — during a PCI-DSS audit — that 18 months of infrastructure changes had no audit trail. Six weeks of manual reconstruction followed. A Harness IaCM pipeline enforces immutable audit records on every run, automatically.

### Maturity Assessment

| Capability | Level 1 — Ad-Hoc | Level 3 — Defined | Level 5 — Optimised |
|---|---|---|---|
| Pipeline coverage | Some resources have pipelines | All apply operations use pipelines | All lifecycle actions (plan/apply/destroy/drift) have pipelines |
| Template usage | No templates | Project-level templates | Org/account-level templates for all governance steps |
| Pipeline-as-code | Pipelines built in UI | YAML defined | Pipelines in Git with PR review — GitOps triggered |
| Conditional execution | Separate apply/destroy pipelines | Stage variables control execution | Dynamic execution based on policy and cost outcomes |

---

## Pillar 2 — Security, Policy & Compliance

> *"Infrastructure code is code. Every change deserves the same security scrutiny as application code — ideally more, since it defines the attack surface."*

### Design Principles

1. **Shift left entirely.** Scan the Terraform plan, not the deployed resource. Post-apply security checks are incident response disguised as governance.

2. **Gate on findings — don't just report them.** CRITICAL and HIGH severity findings block the pipeline. Warning-only scans provide false assurance.

3. **OPA policies are versioned business rules.** Tag enforcement, approved regions, instance size limits, and naming conventions all live as OPA policies in Git — not tribal knowledge.

4. **Secrets never touch pipeline YAML or HCL.** OIDC workload identity is preferred — no stored credentials at all. Vault AppRole for dynamic short-lived tokens where OIDC is unavailable.

### Supported Security Scanning Stack

| Tool | Category | What It Catches | Integration | Type |
|------|----------|-----------------|-------------|------|
| Checkov | IaC Static Analysis | HIPAA, PCI-DSS, SOC 2 misconfigs in Terraform plan | Native Harness IaCM plugin step | OSS |
| Wiz IaC | Cloud Misconfiguration | CIS Benchmarks, NIST CSF, cloud-specific guardrails | Native Harness IaCM plugin step | Commercial |
| tfsec / Trivy | Secrets + SBOM | Hardcoded credentials, open security groups | Plugin step or STO stage | OSS |
| OPA / Conftest | Policy as Code | Tagging, naming, approved regions, instance limits | Native Harness OPA — auto-evaluates on plan/state/workspace | OSS |
| Harness STO | SecOps Orchestration | Aggregates all tools, centralised findings, severity-gated promotion | Dedicated STO stage | Native |
| Gitleaks | Secrets in Git | Credentials committed to IaC repos | Pre-clone step or CI integration | OSS |
| Custom Tools | Custom Scanning | Organisation-specific compliance tooling | Harness STO custom scan integration | Via STO |

### Severity Gates

| Severity | Checkov / Wiz | OPA Policy | Cost Delta | Recommended Action |
|----------|---------------|------------|------------|--------------------|
| CRITICAL | Block pipeline | Block pipeline | >$50k/mo | Override requires CISO approval + rationale log |
| HIGH | Block pipeline | Block pipeline | >$10k/mo | Manager approval + mandatory comment |
| MEDIUM | Warn | Warn | >$5k/mo | Approval note required |
| LOW / Info | Inform | Log only | <$1k/mo | No gate — logged to audit trail |

### Real-World Consequence

> An engineering team pushed an S3 bucket configuration with `public-read` ACL — Checkov would have blocked it at plan time in under 12 seconds. Without IaCM scanning, the bucket sat exposed for 11 days before a third-party audit surfaced it. The remediation cost 3× the prevention cost.

### OPA Policy Examples

**Mandatory tagging:**

```rego
package terraform.tags

required_tags = {"Environment", "Team", "Owner", "CostCentre"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.after != null
  missing := required_tags - {tag | resource.change.after.tags[tag]}
  count(missing) > 0
  msg := sprintf("'%v' is missing required tags: %v", [resource.address, missing])
}
```

**Approved regions only:**

```rego
package terraform.regions

approved = {"us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"}

deny[msg] {
  resource := input.resource_changes[_]
  region := resource.change.after.region
  not approved[region]
  msg := sprintf("'%v' targets unapproved region '%v'", [resource.address, region])
}
```

### Maturity Assessment

| Capability | Level 1 | Level 3 | Level 5 |
|---|---|---|---|
| IaC scanning | No scanning | Single tool (Wiz or Checkov) | Multi-layer: static + secrets + OPA + STO dashboard |
| Scan gates | Warnings only | Block on CRITICAL | Block HIGH, warn MEDIUM, STO trending |
| Secrets management | Variables / hardcoded | Harness Secrets Store | OIDC or Vault AppRole — zero stored credentials |
| Policy as code | No policies | Manual tag enforcement | OPA policies in Git, evaluated on every plan + workspace save |
| Audit trail | Pipeline logs only | Structured scan results | Centralised compliance dashboard with retention |

---

## Pillar 3 — State Management & Reliability

> *"Terraform state is the ground truth of your infrastructure. Treat it with the same care as a production database."*

### Design Principles

1. **Remote state with locking is non-negotiable.** Local state files are a single point of failure. Harness IaCM managed state provides built-in locking, versioning, and encryption — with zero backend infrastructure to maintain.

2. **State is versioned and recoverable in under one hour.** Enable revision history. Know the recovery procedure before an incident — not during one.

3. **Never run state commands from a developer terminal.** `terraform state mv`, `state rm`, and `terraform import` all run through an IaCM pipeline with an approval gate — maintaining the audit trail.

4. **One state file per component × environment.** Sharing state across environments or unrelated resources turns one failed apply into a production incident.

### State Backend Comparison

| Backend | Locking | Versioning | Harness Native | Best For |
|---------|---------|------------|----------------|----------|
| Harness IaCM Managed | ✅ Built-in | ✅ Revision history | ✅ Yes | All new projects — zero backend ops |
| AWS S3 + DynamoDB | ✅ DynamoDB | ✅ S3 versioning | Via connector | AWS-native teams |
| GCS | ✅ Built-in | ✅ Object versioning | Via connector | GCP-native teams |
| Azure Blob | ✅ Lease-based | ✅ Blob versioning | Via connector | Azure-native teams |
| Local file | ❌ None | ❌ None | ❌ No | **Never in production** |

### Workspace Isolation Model

```
Application: PaymentService
│
├── workspace: payment_service_networking_dev
├── workspace: payment_service_networking_stg
├── workspace: payment_service_networking_prod
│
├── workspace: payment_service_compute_dev
├── workspace: payment_service_compute_stg
├── workspace: payment_service_compute_prod
│
└── workspace: payment_service_database_dev
    workspace: payment_service_database_stg
    workspace: payment_service_database_prod

Rule: One state file per (component × environment). Never share state.
```

### Disaster Recovery Runbook

| Scenario | Detection | Recovery | RTO |
|----------|-----------|----------|-----|
| State file corruption | Plan fails with state errors | Restore from last versioned backup, re-run plan | < 1 hour |
| State lock stuck | Timeout on pipeline run | Check lock holder, force-unlock via pipeline (not CLI) | < 30 min |
| Workspace deleted | Pipeline 404 on workspace API | Re-create workspace, import state from backup, drift detect | 2–4 hours |
| Partial apply failure | Apply exits non-zero mid-run | Do NOT re-run blindly — run plan first, inspect diff, apply with targeted changes | 1–2 hours |

### Real-World Consequence

> A team ran `terraform apply` simultaneously from two terminals against the same workspace — no locking, no remote backend. State corruption meant 14 AWS resources were orphaned with no Terraform record. Manual reconciliation took four engineers three days. Harness IaCM managed state prevents this by design.

### Maturity Assessment

| Capability | Level 1 | Level 3 | Level 5 |
|---|---|---|---|
| State backend | Local files | Remote backend (S3/GCS/Azure) | Harness IaCM managed state with locking |
| State isolation | Monolithic state | Per-application state | Per-component per-environment workspace isolation |
| State recovery | No backup | Manual backup | Versioned backend + tested recovery runbook |
| State operations | Manual CLI | CLI with documentation | All state operations via IaCM pipelines with approval |
| Drift handling | Unknown | Manual detection | Automated drift pipelines with alerting and remediation |

---

## Pillar 4 — Module Registry & Workspace Governance

> *"A workspace is the unit of ownership in IaCM. Good workspace design maps directly to team structure, environment topology, and cost allocation."*

### Design Principles

1. **Define once, reuse everywhere via Harness Module Registry.** Platform engineering maintains gold-tier modules (VPC, EKS, RDS, GKE). Teams consume from the registry — they don't write raw resource blocks.

2. **Workspace Templates encode approved defaults.** New workspaces start from a pre-approved template — variables, connectors, policies, and pipeline defaults are already configured.

3. **Workspaces are provisioned by pipelines, not UI clicks.** Name, tags, Git connector, Terraform version, and RBAC assignments are set programmatically.

4. **Naming conventions are enforced, not suggested.** OPA policies reject workspace names that don't conform to the standard scheme.

### Workspace Naming Convention

```
{org}-{team}-{application}-{component}-{environment}-{region}
```

**Examples:**

```
tuk-platform-payments-networking-prod-use1
tuaf-aisadmin-aurora-rds-dev-use1
onetru-vanguard-gke-cluster-nonprod-usc1
```

| Field | Length | Values |
|-------|--------|--------|
| org | 3–6 | tuk, tuaf, onetru, intl |
| team | 2–12 | platform, aisadmin, vanguard |
| app | 2–16 | payments, aurora, gke |
| component | 2–12 | networking, rds, cluster |
| env | 3–5 | dev, stg, prod, test, dr |
| region | 4–5 | use1, usw2, euw1, apse1 |

### Module Governance Tiers

| Tier | Security Review | OPA Compliant | Cost Estimated | Maintained By |
|------|----------------|---------------|----------------|---------------|
| Tier 1 — Gold ⭐ | Full review | ✅ Yes | ✅ Yes | Platform Engineering |
| Tier 2 — Silver | Basic review | ✅ Yes | ⚠️ Partial | Team + Platform co-ownership |
| Tier 3 — Bronze | Self-attested | ⚠️ Partial | ❌ No | Team-owned |
| Unvetted | ❌ None | ❌ No | ❌ No | *Blocked by OPA policy* |

### Recommended Tier 1 Module Catalogue

| Module | Provider | Covers |
|--------|----------|--------|
| tf-aws-vpc | AWS | VPC, subnets, NAT, IGW, flow logs |
| tf-aws-eks | AWS | EKS cluster, node groups, IRSA |
| tf-aws-rds-postgres | AWS | RDS Aurora, parameter groups, encryption |
| tf-aws-s3-secure | AWS | S3 with encryption, versioning, public-access-block |
| tf-gcp-gke | GCP | GKE cluster, node pools, workload identity |
| tf-gcp-cloudsql | GCP | Cloud SQL, encryption, private IP |
| tf-azure-aks | Azure | AKS, managed identity, monitoring |

### RBAC Model

| Role | Plan | Apply (Non-Prod) | Apply (Prod) | Destroy | State Ops |
|------|------|-----------------|-------------|---------|-----------|
| Developer | ✅ | ✅ | ❌ | ❌ | ❌ |
| Team Lead | ✅ | ✅ | ✅ (approval) | ✅ (approval) | ❌ |
| Platform Engineer | ✅ | ✅ | ✅ | ✅ | ✅ (pipeline) |
| Pipeline Service Account | ✅ | ✅ | ✅ | ✅ | ✅ |
| Read-Only / Auditor | ✅ | ❌ | ❌ | ❌ | ❌ |

### Real-World Consequence

> PlayQ reduced infrastructure provisioning from **hours to minutes** after adopting Harness Module Registry — teams stopped reimplementing EKS node group configuration from scratch on every project, eliminating a class of recurring misconfigurations in the process.

### Maturity Assessment

| Capability | Level 1 | Level 3 | Level 5 |
|---|---|---|---|
| Naming convention | No convention | Documented but inconsistent | Enforced via OPA policy on workspace create |
| RBAC | Shared admin access | Role-based per project | Workspace-scoped RBAC with least-privilege service accounts |
| Workspace provisioning | Manual UI creation | Documented process | Self-service via IaCM pipeline or catalog |
| Lifecycle management | No process | Ad-hoc decommissioning | Automated quarterly reviews with cost-based alerting |
| Tagging | No workspace tags | Some tags | Mandatory tags enforced, untagged workspaces auto-flagged |

---

## Pillar 5 — Cost & FinOps Integration

> *"The cheapest infrastructure is the infrastructure you didn't provision by accident. The second cheapest is the infrastructure you right-sized before it went live."*

### Design Principles

1. **Cost is a pre-apply gate.** Harness IaCM integrates Infracost natively — engineers see the monthly cost delta in the approval step before a single resource is provisioned.

2. **OPA enforces budget thresholds as policy.** A change exceeding $10k/month blocks the pipeline automatically — no separate FinOps tool escalation required.

3. **Harness CCM closes the feedback loop.** When CCM detects a cost anomaly, the first signal surfaces which IaCM pipeline ran nearest to the spike.

4. **Ephemeral environments are never always-on.** Dev and test workspaces use Harness AutoStopping or scheduled destroy pipelines. Idle environments running nights and weekends typically represent 60–70% waste.

### Cost Estimation in the Pipeline

```
terraform plan output
        │
        ▼
┌─────────────────┐     ┌───────────────────────────────┐
│  Infracost CLI  │────►│         Cost Report            │
│  (Plugin step)  │     │  Monthly cost (before): $1,240 │
└─────────────────┘     │  Monthly cost (after):  $3,890 │
                        │  Delta:                +$2,650  │
                        │                                 │
                        │  Top cost drivers:              │
                        │  • aws_instance.app  +$1,800/mo │
                        │  • aws_rds_cluster   +$850/mo   │
                        └───────────────────────────────┘
                                        │
                                        ▼
                              ┌─────────────────┐
                              │   Budget Gate   │
                              │  >$5k → block   │
                              │  >$1k → warn    │
                              └─────────────────┘
```

### Required Cost Allocation Tags

| Tag Key | Example Value | FinOps Purpose |
|---------|---------------|----------------|
| Environment | production | Filter prod vs non-prod spend |
| Team | platform-engineering | Team-level chargeback |
| Application | payments-service | Per-application cost attribution |
| CostCentre | CC-4712 | Finance chargeback code |
| Owner | sam@company.com | Accountability — alert recipient |
| ManagedBy | harness-iacm | Identify IaCM-provisioned resources in CCM |
| Workspace | tuk-platform-payments-prod-use1 | Workspace-level cost tracking |

Enforce via OPA:

```rego
package terraform.cost_tags

required = {"Environment","Team","Application","CostCentre","Owner","ManagedBy"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.after != null
  missing := required - {k | resource.change.after.tags[k]}
  count(missing) > 0
  msg := sprintf("'%v' missing cost allocation tags: %v", [resource.address, missing])
}
```

### Ephemeral Environment TTL Strategy

| Environment Type | Expected Lifetime | Recommended Mechanism |
|-----------------|-------------------|----------------------|
| Feature branch dev | Hours to days | Harness AutoStopping + destroy pipeline on PR close |
| Sprint testing | Days to weeks | Scheduled destroy pipeline (business hours only) |
| Demo / sandbox | Days | AutoStopping with idle detection |
| Integration testing | Pipeline duration | Destroy stage at end of CI pipeline |
| Staging | Permanent | AutoStopping on out-of-hours (20:00–06:00) |
| Production | Permanent | Right-sizing recommendations via CCM |

### CCM ↔ IaCM Feedback Loop

```
Harness CCM (FinOps)
│
├─► Cost Anomaly Detected
│       └─► Investigate: which IaCM pipeline ran at T-1h?
│
├─► Recommendation: Downsize instance
│       └─► Create IaC PR → pipeline apply → cost reduction realised
│
├─► Budget Alert: 85% consumed
│       └─► Notify workspace owner → review pending applies
│
└─► Cost Attribution (tag-based)
        └─► Workspace tags propagate to CCM Perspectives
            └─► Cost breakdown by team / app / environment
```

### Real-World Consequence

**Without IaCM FinOps integration:** An engineer resizes an RDS cluster from `db.t3.medium` to `db.r6g.2xlarge` for "a quick performance test." No cost estimate, no approval gate. The instance runs over a long weekend. Finance receives a $28,000 surprise on the next bill.

**With IaCM FinOps integration:** The pipeline shows a +$19,200/month delta at the approval step. The approver adds a comment: *"approved for 3-day test — destroy pipeline scheduled Friday 18:00."* Cost impact is logged, attributable, and time-bounded.

### Maturity Assessment

| Capability | Level 1 | Level 3 | Level 5 |
|---|---|---|---|
| Cost estimation | No estimation | Manual Infracost run | Infracost in every apply pipeline with budget gate |
| Cost tagging | No tags or inconsistent | Partial tagging | OPA-enforced mandatory tags on all resources |
| Ephemeral environments | Always-on | Manual teardown | AutoStopping + scheduled destroy pipelines |
| Budget integration | No budgets | CCM budgets exist | Budget alerts linked to workspace apply gates |
| CCM ↔ IaCM feedback | Manual | Monthly review | Automated: anomaly → IaCM investigation → PR |

---

## Pillar 6 — Developer Experience & Enablement

> *"The best governance is governance that developers don't have to think about — it's built into the platform, invisible when things are right, and clearly instructive when they're not."*

### Design Principles

1. **Self-service workspace provisioning under 15 minutes.** Harness IDP Service Catalog form → provisioning pipeline → workspace, RBAC, backend, monitoring, and pipeline — fully configured, zero platform engineer involvement.

2. **Golden-path templates encode the guardrails invisibly.** Security scans, approval gates, and cost estimation are embedded in the template. Teams benefit from governance without being aware of implementing it.

3. **All feedback in one pane.** Plan diff, Wiz findings, cost delta, and OPA violations surface in the Harness pipeline UI — or as a PR comment. Not across five separate tool portals.

4. **Adoption is measured as a platform KPI.** Workspace count, scan coverage, drift rate, cost attribution accuracy, and self-service percentage are reviewed monthly — not assumed to be improving.

### The Golden Path

```
Developer Experience          Platform Engineering Provides
──────────────────            ──────────────────────────────
1. Request workspace    ───►  Catalog form → provisioning pipeline
                              Creates: workspace, RBAC, backend,
                              pipeline, monitoring, tags

2. Write Terraform code ───►  Golden path module library
   (using approved modules)   Pre-vetted: VPC, EKS, RDS, GKE, etc.

3. Open Pull Request    ───►  Harness Code / GitHub integration
   (triggers plan pipeline)   Runs: plan → scan → cost estimate
                              Posts results as PR comment

4. Review plan results  ───►  Harness Pipeline UI
   (in UI or PR)              Shows: plan diff, Wiz findings,
                              cost delta, policy violations

5. Merge PR → apply     ───►  GitOps trigger
   (auto or manual)           Runs: full lifecycle pipeline
                              Approval gate with plan diff

6. Monitor workspace    ───►  Harness CCM + Drift pipeline
                              Alerts on drift, budget, anomalies
```

### Target Platform KPIs

| KPI | Target | Review Cadence |
|-----|--------|----------------|
| Workspace scan coverage | 100% | Monthly |
| Drift rate (production) | < 5% | Weekly automated alert |
| MTTR — drift to remediation | < 4 hours | Monthly |
| Cost attribution accuracy | > 95% tagged resources | Monthly |
| Self-service provisioning % | > 80% of new workspaces | Monthly |
| Mean time to new workspace | < 15 minutes | Monthly |

### Harness IDP Integration

```
Harness IDP Service Catalog
│
├─► Software Template: "Create IaCM Workspace"
│       Input:  team, app, environment, cloud, region
│       Action: triggers provisioning pipeline
│       Output: workspace URL, pipeline URL, runbook link
│
├─► Scorecard: "IaCM Best Practices"
│       Checks: security scan, drift detection, tagging, approval gates
│       Score:  0–100, visible on every component page
│
└─► TechDocs: "IaCM Runbooks"
        Platform docs, migration guides, module docs
        Auto-synced from Git
```

### Real-World Consequence

> When workspace provisioning requires a Jira ticket and a 5-day SLA, engineers find workarounds — personal AWS accounts, untracked Terraform local state, manual console provisioning. **Self-service is not a convenience feature. It is a security requirement.** Ungoverned infrastructure emerges when governed infrastructure is too slow.

### Maturity Assessment

| Capability | Level 1 | Level 3 | Level 5 |
|---|---|---|---|
| Workspace provisioning | Manual + ticket | Documented process, <5 day SLA | Self-service catalog, <15 min automated provisioning |
| Module library | No shared modules | Shared modules in Git | Tiered governance model, Tier 1 gold modules enforced |
| Developer feedback | Pipeline logs only | Scan results in Harness UI | PR comments with plan, cost, scan, policy in one view |
| Platform KPIs | Not measured | Quarterly review | Monthly dashboard, automated alerts on regression |
| IDP integration | None | Basic catalog | Full scorecard, templates, TechDocs |

---

## Full Maturity Scorecard

Use this scorecard to assess your overall IaCM Well-Architected posture. Score each pillar 1–5, multiply by weight, and sum for a weighted total out of 5.0.

| Pillar | Weight | Your Score (1–5) | Weighted Score |
|--------|--------|-----------------|----------------|
| Pipeline Design & Lifecycle | 20% | ___ | ___ |
| Security, Policy & Compliance | 25% | ___ | ___ |
| State Management & Reliability | 20% | ___ | ___ |
| Module Registry & Workspace Governance | 15% | ___ | ___ |
| Cost & FinOps Integration | 10% | ___ | ___ |
| Developer Experience & Enablement | 10% | ___ | ___ |
| **Total** | **100%** | — | **___/5.0** |

### Maturity Bands

| Score | Band | Description |
|-------|------|-------------|
| 1.0 – 1.9 | **Foundational** | IaCM module enabled, some pipelines exist, manual processes dominate |
| 2.0 – 2.9 | **Developing** | Core lifecycle pipelines running, basic approval gates, some templating |
| 3.0 – 3.9 | **Defined** | Consistent pipeline patterns, security scanning, drift detection, tagging |
| 4.0 – 4.5 | **Managed** | OPA policies, cost estimation, self-service, measured KPIs |
| 4.6 – 5.0 | **Optimised** | Auto-remediation, FinOps feedback loop, SBOM, full self-service |

---

## Prioritised Improvement Roadmap

### Phase 1 — Harden the Foundation (Days 0–30)

- **Action 1:** Apply account-level scan template (Checkov + Wiz) to all pipelines lacking it — 2-hour effort, zero new tooling.
- **Action 2:** Add `IaCMApproval` gates with `autoApprove: false` and `timeout: 24h` to every destroy pipeline. No exceptions.
- **Action 3:** Create an account-level OPA policy enforcing `Environment`, `Team`, `Owner`, and `CostCentre` on all resources. Block on non-compliance.
- **Action 4:** Enable daily drift detection (08:00 UTC) for every production workspace. Alert to Slack / PagerDuty when drift is detected.
- **Action 5:** Add Infracost as a plugin step to the account-level scan template. All pipelines referencing it get cost estimation automatically. Set a $5k/month gate.

### Phase 2 — Add Governance (Days 30–90)

- **Action 6:** Create org-level pipeline templates for all business units. One template change covers all pipelines organisation-wide.
- **Action 7:** Replace long-lived AWS/GCP credentials with OIDC / Workload Identity Federation on all cloud connectors — zero stored credentials.
- **Action 8:** Migrate standalone Wiz shell-script integrations to native Harness STO stages for a centralised findings dashboard and historical trending.
- **Action 9:** Populate Module Registry with Tier 1 gold modules for the top 10 infrastructure patterns. Enforce via OPA — unvetted module sources blocked at plan time.

### Phase 3 — Optimise & Scale (Days 90–180)

- **Action 10:** Build a Harness IDP Service Catalog template for end-to-end workspace provisioning. Target SLA: < 15 minutes.
- **Action 11:** Connect Harness CCM anomaly detection to IaCM pipeline run history. When a cost spike is detected, automatically surface which pipeline ran nearest to the spike time.
- **Action 12:** Implement AutoStopping and scheduled destroy pipelines for all non-production environments.
- **Action 13:** Extend drift pipelines to auto-create Jira tickets with the drift diff and optionally trigger an approval-gated remediation apply pipeline.
- **Action 14:** Publish a monthly platform KPI dashboard covering Workspace Coverage, Drift Rate, MTTR, Cost Attribution, and Self-Service %.

---

## Appendix A — Real-World Patterns Reference

| Pattern | Observed In | Recommendation |
|---------|-------------|----------------|
| Unified lifecycle pipeline (TF_Action variable) | TU Africa / AISADMINIAC | Recommended for all new workspaces |
| Org-level pipeline template | TU UK | Mandatory for all orgs |
| Vault AppRole dynamic credentials | OneTru / Vanguard | Recommended where Vault is available |
| Dynamic workspace naming from env variables | OneTru / Vanguard | Best practice for multi-env deployments |
| Account-level security scan template | TU Africa / TU UK | Mandatory — centralise all security controls |
| Dedicated drift pipeline (schedulable) | TU UK | Recommended alongside embedded drift step |
| CloudFormation → Terraform migration pipeline | TU Africa | Reference pattern for all legacy migrations |
| State file migration pipeline | TU Africa | Mandatory — no manual state ops |

---

## Appendix B — Harness IaCM Step Quick Reference

| Step Type | Command | Purpose |
|-----------|---------|---------|
| IACMTerraformPlugin | `init` | Initialise Terraform, configure backend |
| IACMTerraformPlugin | `plan` | Generate execution plan, output plan JSON |
| IACMTerraformPlugin | `apply` | Apply the plan, provision resources |
| IACMTerraformPlugin | `destroy` | Destroy all resources in workspace |
| IACMTerraformPlugin | `detect-drift` | Compare state vs live infrastructure |
| IACMApproval | — | Human approval gate (shows plan diff + cost delta) |
| Policy | — | OPA policy evaluation |
| Plugin | `infracost` | Cost estimation from plan JSON |
| ShellScript / Run | bash | Pre/post processing, credential setup |
| STO stage | — | Security test orchestration and findings dashboard |

---

*The IaCM Well-Architected Framework is a living document. Contribute improvements via pull request to the platform engineering repository.*

*Harness IaCM Centre of Excellence · May 2026 · Confidential*
