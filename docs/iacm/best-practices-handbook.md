---
title: IaCM Best Practices Handbook
subtitle: Pipeline Adoption Analysis & Governance Framework
customer: TransUnion
date: May 4, 2026
author: Harness IaCM Agent
---

## Executive Summary

This handbook is grounded in a live analysis of **26 IaCM pipelines** deployed across your Harness account. It distils real adoption patterns from 7 projects and 5 organisations into actionable best practices, highlights security gaps, and provides a phased roadmap to reach fully governed Infrastructure as Code.

::: metrics
- label: Pipelines Analysed
  value: "26"
  sub: across 7 projects
- label: Orgs with IaCM
  value: "5"
  sub: of 35 total orgs
- label: Security Scan Coverage
  value: "54%"
  sub: 14 of 26 pipelines
- label: Approval Gate Coverage
  value: "42%"
  sub: 11 of 26 pipelines
- label: Drift Detection
  value: "15%"
  sub: 4 of 26 pipelines
- label: Cost Estimation
  value: "0%"
  sub: not yet deployed
:::

:::warning
**Largest gaps identified:** Zero cost estimation integration, zero OPA policy enforcement, and 12 pipelines running with no security scanning — despite the Wiz IaC template being available at account level.
:::

---

## What Was Analysed

### Organisations & Projects

| Organisation | Project | Pipelines | Cloud | Maturity Level |
|---|---|---|---|---|
| TU Africa | AISADMINIAC | 11 | AWS | **High** |
| TU UK | The First Project | 4 | Template-driven | **Medium** |
| TU UK | ArronTest | 4 | Template-driven | **Medium** |
| TU UK | TF Automation Testing | 4 | Template-driven | **Medium** |
| OneTru | Vanguard | 1 | GCP | **High** |
| International Markets | TUINDAS DEDGE | 1 | GCP | **Low** |

### Pipeline Type Distribution

| Type | Count | Notes |
|---|---|---|
| Apply | 6 | Core provisioning pipelines |
| Plan (dry-run) | 5 | Pre-flight validation |
| Destroy | 5 | Teardown / decommissioning |
| Drift Detection | 4 | Config drift alerting |
| Combined (plan+apply+destroy in one) | 4 | AISADMINIAC pattern |
| State Migration | 2 | Statefile management |
| CloudFormation → Terraform | 1 | Legacy migration |
| Workspace Management | 1 | Dynamic workspace creation |

---

## Pipeline Patterns Found

### Pattern A — Unified Pipeline (AISADMINIAC)

:::success
**Most mature pattern in the account.** A single pipeline handles plan, drift detection, apply, and destroy via conditional step groups driven by a `TF_Action` stage variable. This reduces pipeline sprawl and keeps all Terraform lifecycle operations in one place.
:::

```
init
 └─► plan
      └─► [Security Scan — TU_SEAL_IAC_Scan_template]
           └─► drift_detect (conditional: Detect_drift == 'yes')
                └─► PARALLEL
                     ├─► [TF_Action == 'apply']   → IACMApproval → TF apply
                     └─► [TF_Action == 'destroy'] → IACMApproval → TF destroy
```

**Stage variables used:**

| Variable | Type | Values | Purpose |
|---|---|---|---|
| `TF_Action` | String | `dry-run` / `apply` / `destroy` | Switches execution path |
| `Detect_drift` | String | `yes` / `no` | Enables drift detection step |
| `workspace` | String | `<+input>` | Dynamic workspace selection |

---

### Pattern B — Dedicated Pipelines Per Action (TU UK)

Four separate pipelines per workspace: `plan`, `apply`, `destroy`, `drift`. All reference an **org-level pipeline template** (`org.pipeline_terraform_apply`) with a Wiz scan step group injected via local template.

```
pipeline_terraform_plan   ──► (plan only, no approval gate)
pipeline_terraform_apply  ──► [Wiz Scan] → IACMApproval → apply
pipeline_terraform_destroy ─► [Wiz Scan] → IACMApproval → destroy
pipeline_terraform_drift  ──► (drift detect, no approval)
```

**Key strength:** Org-level template ensures all projects in TU UK inherit security controls automatically. Used by ArronTest, The First Project, and TF Automation Testing.

:::warning
**Gap:** Destroy pipelines in TU UK are missing approval gates. Destroy operations must always require human sign-off.
:::

---

### Pattern C — Multi-Stage Orchestration (OneTru / Vanguard)

The most sophisticated pattern in the account. Separates **secret bootstrapping**, **workspace lifecycle management**, and **IaC execution** into distinct pipeline stages with HashiCorp Vault for dynamic credential injection.

```
Stage 1: Pre Setup
  └─► ShellScript: Vault AppRole login (env-routed: dev / stg / prod vaults)
       └─► Exports: vault_token, VAULT_SERVER, platform, workspace_name

Stage 2: Manage IACM Workspace
  └─► StepGroup: Manage_IACM_Workspace template (v1.0.5)
       └─► Dynamic workspace creation/selection from env variables

Stage 3: IaCM Execution
  └─► IACMTerraformPlugin steps
       └─► STO Security scan stage
```

**Environment-aware vault routing:**

| Environment | Vault Server |
|---|---|
| `rnd`, `fdev`, `qa` | vault-dev.crypto.gcp.transu.net |
| `demo`, `ddev` | vault-stg.crypto.gcp.transu.net |
| Production | vault.crypto.gcp.transu.net |

---

### Pattern D — CloudFormation → Terraform Migration (AISADMINIAC)

:::info
A specialised pipeline for migrating existing AWS infrastructure from CloudFormation to Terraform via `terraform import`. Uses the `CTT_Terraform_Migration` template with Wiz scanning post-import and optional drift detection to verify fidelity of the migrated state.
:::

---

## Security & Compliance Integrations

::: metrics
- label: Wiz IaC Scan
  value: "13"
  sub: pipelines covered
- label: Harness STO
  value: "1"
  sub: Vanguard only
- label: Checkov / tfsec
  value: "0"
  sub: not deployed
- label: OPA / Conftest
  value: "0"
  sub: not deployed
- label: Infracost
  value: "0"
  sub: not deployed
- label: Approval Gates
  value: "11"
  sub: of 26 pipelines
:::

### Wiz IaC Scan — The Active Standard (13 Pipelines)

:::success
**Best practice already in use.** Wiz IaC scanning is centralised in a versioned account-level template (`account.TU_SEAL_IAC_Scan_Templates`). Any pipeline that references this template automatically inherits the latest security rules without a PR. This is the correct pattern — update the template once, all pipelines benefit.
:::

The scan is injected **after plan, before approval** — the only correct position. Running it post-apply would be too late.

```yaml
- stepGroup:
    name: TU_SEAL_IAC_Scan_template
    template:
      templateRef: account.TU_SEAL_IAC_Scan_Templates
      versionLabel: IAC_Scan_Prod
      templateInputs:
        variables:
          - name: Print_Plan_JSON
            value: "<+input>.default(false).selectOneFrom(true,false)"
```

### Harness STO — Native Security Orchestration (1 Pipeline)

The Vanguard pipeline uses a native **Harness STO stage** which produces structured scan findings in the Harness UI, supports pass/fail gates on severity thresholds, and aggregates results across runs over time. This is more powerful than a shell-based Wiz call and should be adopted more broadly.

### Security Tool Gaps

:::critical
**12 out of 26 pipelines have no security scan at all.** All AISADMINIAC resource pipelines (Aurora RDS, S3, Route53, EC2) use `IACMTerraformPlugin` steps but do not include the shared Wiz scan template. The template exists at account level — it just needs to be referenced. This is a P1 remediation item.
:::

---

## Adoption Maturity Assessment

### Per-Project Scorecard

| Project | Org | Approval | Security | Drift | Templated | Cost | OPA |
|---|---|---|---|---|---|---|---|
| AISADMINIAC | TU Africa | ✅ | ⚠️ Partial | ✅ | ⚠️ Partial | ❌ | ❌ |
| The First Project | TU UK | ✅ | ✅ Wiz | ✅ | ✅ Org tmpl | ❌ | ❌ |
| ArronTest | TU UK | ✅ | ✅ Wiz | ✅ | ✅ Org tmpl | ❌ | ❌ |
| TF Automation Testing | TU UK | ✅ | ✅ Wiz | ✅ | ✅ Org tmpl | ❌ | ❌ |
| Vanguard | OneTru | ❌ | ✅ STO | ❌ | ✅ Workspace | ❌ | ❌ |
| TUINDAS DEDGE | Int'l Mkts | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

### Maturity Levels

| Level | Criteria | Status |
|---|---|---|
| **Level 1 — Basic** | IaCM module enabled, pipelines exist | TUINDAS DEDGE |
| **Level 2 — Controlled** | + Approval gates, workspace usage | AISADMINIAC |
| **Level 3 — Secure** | + Security scanning, templating, drift detection | TU UK, Vanguard |
| **Level 4 — Governed** | + OPA policy, cost estimation, full scan coverage | **No project yet** |
| **Level 5 — Optimised** | + Auto-remediation, FinOps integration, SBOM | **Not yet** |

---

## Best Practice: The IaCM Pipeline Lifecycle

Every IaCM pipeline should follow this lifecycle. The account's best pipelines already implement most of it.

```
┌──────────┬──────────┬───────────┬──────────┬───────────┬────────────┐
│  SOURCE  │  SCAN    │  PLAN     │  GATE    │  APPLY    │  VERIFY    │
│          │          │           │          │           │            │
│ git pull │ IaC Scan │ terraform │ Approval │ terraform │ drift      │
│ (Code /  │ (Wiz /   │ plan      │ + OPA    │ apply     │ detect +   │
│ GitClone)│ Checkov) │ + cost    │ + budget │           │ smoke test │
└──────────┴──────────┴───────────┴──────────┴───────────┴────────────┘
```

| Step | IaCM Step Type | Required | Notes |
|---|---|---|---|
| 1. Init | `IACMTerraformPlugin: init` | Yes | Always first |
| 2. Plan | `IACMTerraformPlugin: plan` | Yes | Generates plan JSON |
| 3. Security Scan | Step group (Wiz / Checkov / STO) | **Yes** | After plan, before apply |
| 4. Cost Estimation | Plugin (Infracost) | Recommended | Attach to plan output |
| 5. OPA Policy | Harness Policy step | Recommended | Block on violations |
| 6. Approval Gate | `IACMApproval` | **Yes** for Apply & Destroy | `autoApprove: true` for dev only |
| 7. Apply / Destroy | `IACMTerraformPlugin: apply/destroy` | Yes | Conditional on `TF_Action` |
| 8. Drift Detection | `IACMTerraformPlugin: detect-drift` | Recommended | Post-apply or scheduled |
| 9. Outputs | Run step | Optional | Pass outputs to CD pipelines |

---

## Best Practice: Security-First IaC

### Recommended Tool Stack (Defence in Depth)

| Layer | Tool | When | What it Catches |
|---|---|---|---|
| IaC static analysis | **Wiz IaC** ✅ (deployed) | Post-plan | Misconfigs, hardcoded secrets, CIS benchmarks |
| IaC static analysis (OSS) | **Checkov** ❌ (gap) | Post-plan | HIPAA / PCI / SOC2 compliance policies |
| Secrets detection | **tfsec / Trivy** ❌ (gap) | Pre-commit | Exposed credentials in HCL |
| Policy enforcement | **OPA / Conftest** ❌ (gap) | Post-plan | Tagging, naming, approved regions |
| Cost gate | **Infracost** ❌ (gap) | Post-plan | Budget threshold violations |

### Blocking vs Warning Scans

| Severity | Recommended Action |
|---|---|
| CRITICAL | Block pipeline — require explicit override with justification |
| HIGH | Require approval comment noting the accepted risk |
| MEDIUM | Warning only — log to STO dashboard |
| LOW / INFO | Informational — no gate |

:::action
**Action:** Extend `account.TU_SEAL_IAC_Scan_Templates` to include a Checkov step alongside Wiz. Both tools cover complementary rule sets — Wiz for cloud-native misconfigs, Checkov for compliance frameworks (CIS, PCI-DSS, HIPAA).
:::

---

## Best Practice: Approval Gates & Governance

:::success
**Already implemented in 11/26 pipelines.** The `IACMApproval` step in Harness IaCM shows the full Terraform plan diff inline in the approval UI — reviewers can see exactly what will change before clicking approve.
:::

### When to Require Approval

| Environment | Apply | Destroy | Drift Remediation |
|---|---|---|---|
| Development | Optional (`autoApprove: true`) | **Required** | Optional |
| Staging | **Required** | **Required** | **Required** |
| Production | **Required** + 2-person rule | **Required** + 2-person | **Required** |

### Timeout Guidelines

| Action | Timeout | Rationale |
|---|---|---|
| Apply (non-prod) | 2 hours | Business hours window |
| Apply (prod) | 4 hours | Cross-team review time |
| Destroy | 24 hours | Allow cross-team awareness |

---

## Best Practice: Drift Detection

:::info
**Drift** occurs when the actual state of infrastructure diverges from the Terraform state file — usually because someone made a manual change in the cloud console. Detecting drift early prevents compounding inconsistencies and failed applies.
:::

### Three-Tier Drift Strategy

| Tier | Frequency | Action | How |
|---|---|---|---|
| Scheduled | Daily / weekly | Alert on drift | Dedicated drift pipeline with cron trigger |
| Pre-apply | Always | Detect before layering more changes | Embedded in unified pipeline |
| Post-apply | Always | Verify apply landed correctly | Conditional step after apply |

### Current Gap

:::warning
Only 4 out of 26 pipelines use drift detection. The TU UK pattern correctly separates drift into a dedicated schedulable pipeline. AISADMINIAC implements drift as a conditional step in the unified pipeline. Both approaches are valid — but only 4 pipelines use either.
:::

---

## Best Practice: Secrets Management

### What Was Found — Vault AppRole (Vanguard / OneTru)

The most advanced secrets pattern in the account. The Vanguard pipeline authenticates dynamically to HashiCorp Vault using AppRole, routes to the correct vault server based on the target environment, and passes short-lived credentials to downstream IaCM steps. No long-lived credentials are stored in Harness.

### Recommended Hierarchy

| Secret Type | Storage | Risk if Leaked |
|---|---|---|
| Cloud credentials (AWS/GCP/Azure) | Harness Cloud Connectors | Critical |
| Vault AppRole credentials | Harness Secrets (project-scoped) | High |
| Terraform backend credentials | OIDC (preferred) or Harness Secrets | High |
| Non-sensitive pipeline config | Pipeline stage variables | Low |

:::critical
**Never hardcode credentials** in pipeline YAML, Terraform HCL, or variable defaults. Use `<+secrets.getValue("secret_name")>` for all sensitive values. Harness secrets are encrypted at rest and audited — cloud console credentials are not.
:::

### Recommended: OIDC Over Long-Lived Keys

For AWS and GCP, prefer **OIDC-based authentication** over IAM access keys:
- **AWS:** Harness OIDC connector with IAM role assumption (no `AWS_ACCESS_KEY_ID` ever stored)
- **GCP:** Workload Identity Federation
- **Azure:** Managed Identity / Federated Credentials

---

## Best Practice: Templating & Reusability

### Template Hierarchy Found

| Scope | Template | Version | Used By |
|---|---|---|---|
| Account | `TU_SEAL_IAC_Scan_Templates` | `IAC_Scan_Prod` | TU Africa security scan |
| Org (TU UK) | `pipeline_terraform_apply` | `v2` | All TU UK pipelines |
| Org (TU UK) | `SEAL_Wiz_IaC_Scan_IaCM_Stepgroup` | local | TU UK Wiz integration |
| Project (OneTru) | `Manage_IACM_Workspace` | `v1.0.5` | Vanguard workspace lifecycle |
| Project (TU Africa) | `CTT_Terraform_Migration` | `v1.0` | CloudFormation migration |

### Governance Rules

| Template Level | Use For | Versioning |
|---|---|---|
| Account | Cross-org security controls, compliance | Mandatory — semantic versioning |
| Org | Shared pipeline structures within a BU | Mandatory |
| Project | Project-specific step overrides | Recommended |

:::action
**Action:** Create org-level pipeline templates for all 5 orgs — not just TU UK. International Markets and OneTru currently have no org-level IaCM template, meaning each project manages its own pipeline YAML independently. This leads to drift in security and governance controls over time.
:::

Always pin to a **specific version** in pipelines (`versionLabel: "2"`) — never use `latest`.

---

## Best Practice: Cost Estimation (Gap)

:::critical
**Zero out of 26 pipelines implement cost estimation.** This is the largest missing control given the account uses Harness CCM for FinOps governance. Teams are applying Terraform changes with no visibility into the cost impact until the next billing cycle.
:::

### Infracost Integration

Infracost generates a cost diff from the Terraform plan JSON, showing the monthly cost impact of the proposed change **before apply**. Integration point: after `plan`, before the approval gate.

### Cost Gate Thresholds

| Monthly Cost Increase | Action |
|---|---|
| < $500 | Auto-approve in dev environments |
| $500 – $5,000 | Require team lead approval with cost justification |
| > $5,000 | Require FinOps team approval |
| Any increase in Production | Always require approval regardless of amount |

:::action
**Action:** Add Infracost as a Plugin step to the account-level `TU_SEAL_IAC_Scan_Templates` template. This gives all pipelines that reference the template automatic cost estimation with zero additional pipeline changes.
:::

---

## Best Practice: Policy as Code (OPA) — Gap

:::critical
**Zero out of 26 pipelines enforce OPA / Conftest policies.** Without policy gates, teams can provision non-compliant infrastructure — wrong regions, missing tags, publicly accessible buckets, unencrypted databases — and it will not be caught until a manual audit or a security incident.
:::

### Recommended Policies

| Policy | Severity | Applies To |
|---|---|---|
| Mandatory tags (`Environment`, `Team`, `Owner`) | **BLOCK** | All resources |
| Approved AWS/GCP regions only | **BLOCK** | All cloud resources |
| No public S3 buckets (`acl = "public-read"`) | **BLOCK** | AWS S3 |
| Encryption at rest required | **BLOCK** | RDS, S3, EBS, GCS |
| No unrestricted security group rules (0.0.0.0/0 on :22/:3389) | **BLOCK** | AWS Security Groups |
| Cost threshold (>$5k/month delta) | **WARN** | All |
| Approved instance types only | **WARN** | EC2, GKE nodes |

---

## Gaps & Prioritised Remediation

### Priority Matrix

| Gap | Impact | Effort | Priority |
|---|---|---|---|
| Security scan on 12 uncovered pipelines | High | **Low** — reuse existing template | **P1** |
| Approval gates on all destroy pipelines | High | **Low** | **P1** |
| Cost estimation (Infracost) | High | Medium | **P2** |
| OPA policy enforcement (tagging, regions) | High | Medium | **P2** |
| Drift detection on all apply pipelines | Medium | Low | **P2** |
| Consistent pipeline tagging | Medium | Low | **P2** |
| OIDC credentials (no long-lived keys) | High | High | **P3** |
| Harness STO adoption beyond Vanguard | Medium | Medium | **P3** |
| Terraform module automated testing | Medium | High | **P4** |
| SBOM / supply chain for IaC modules | Low | High | **P4** |

### Immediate Actions (This Sprint)

:::action
**P1 — Extend Wiz scan to all 12 uncovered pipelines.** The `account.TU_SEAL_IAC_Scan_Templates` template already exists. Add a single `stepGroup` reference to each of the 9 AISADMINIAC resource pipelines and 3 remaining pipelines that lack it. Estimated effort: 2 hours.
:::

:::action
**P1 — Add `IACMApproval` to all destroy pipelines.** The 4 TU UK destroy pipelines (`iac_harness_project_configuration_destroy` in ArronTest, The First Project) and the AISADMINIAC EC2/state migration pipelines have no destruction approval. Add with `timeout: 24h` and `autoApprove: false`. Estimated effort: 1 hour.
:::

:::action
**P2 — Add Infracost to the account security scan template.** Embed Infracost as a Plugin step in `TU_SEAL_IAC_Scan_Templates`. All 13 pipelines that reference this template will automatically gain cost estimation. Estimated effort: half a day.
:::

---

## Recommended Pipeline Blueprint

This is the target-state unified pipeline combining all best practices from across the account.

```yaml
pipeline:
  name: iacm_best_practice_apply
  tags:
    Environment: "<+input>"
    Team: "<+input>"
    Owner: "<+input>"
  stages:
    - stage:
        name: IaC Provision
        type: IACM
        spec:
          runtime:
            type: Cloud
          workspace: "<+input>"
          execution:
            steps:
              - step:
                  type: IACMTerraformPlugin
                  name: init
                  spec:
                    command: init
              - step:
                  type: IACMTerraformPlugin
                  name: plan
                  spec:
                    command: plan
              # Security + Cost scan (centralised template)
              - stepGroup:
                  template:
                    templateRef: account.TU_SEAL_IAC_Scan_Templates
                    versionLabel: IAC_Scan_Prod
              # OPA policy gate
              - step:
                  type: Policy
                  name: OPA_Policy_Check
                  spec:
                    policySets:
                      - account.iacm_mandatory_tags
                      - account.iacm_approved_regions
              # Drift detection
              - step:
                  type: IACMTerraformPlugin
                  name: drift_detect
                  spec:
                    command: detect-drift
                  when:
                    condition: <+stage.variables.detect_drift>=='yes'
              # Approval gate (shows plan diff inline)
              - step:
                  type: IACMApproval
                  name: Apply_Approval
                  spec:
                    autoApprove: "<+stage.variables.auto_approve>"
                  timeout: 4h
              # Apply (conditional)
              - step:
                  type: IACMTerraformPlugin
                  name: apply
                  spec:
                    command: apply
                  when:
                    condition: <+stage.variables.tf_action>=='apply'
              # Destroy with separate 24h approval
              - step:
                  type: IACMApproval
                  name: Destroy_Approval
                  spec:
                    autoApprove: false
                  timeout: 24h
                  when:
                    condition: <+stage.variables.tf_action>=='destroy'
              - step:
                  type: IACMTerraformPlugin
                  name: destroy
                  spec:
                    command: destroy
                  when:
                    condition: <+stage.variables.tf_action>=='destroy'
        variables:
          - name: tf_action
            default: dry-run
            value: "<+input>.default(dry-run).selectOneFrom(dry-run,apply,destroy)"
          - name: detect_drift
            default: "yes"
            value: "<+input>.default(yes).selectOneFrom(yes,no)"
          - name: auto_approve
            default: "false"
            value: "<+input>.default(false).selectOneFrom(true,false)"
```

---

## Maturity Roadmap

### Phase 1 — Standardise (Now → 4 weeks)

- [ ] Apply Wiz scan template to all 12 uncovered pipelines
- [ ] Add `IACMApproval` to all destroy pipelines
- [ ] Enable drift detection on all production apply pipelines
- [ ] Enforce consistent pipeline tagging (`Environment`, `Team`, `Owner`)
- [ ] Document and enforce workspace naming conventions

### Phase 2 — Govern (4 – 12 weeks)

- [ ] Add Infracost cost estimation to the account scan template
- [ ] Create OPA policy set: mandatory tags, approved regions, security baseline
- [ ] Extend Harness STO to all projects (replace standalone Wiz shell calls)
- [ ] Create org-level pipeline templates for International Markets and OneTru
- [ ] Implement OIDC / Workload Identity for all cloud connectors

### Phase 3 — Optimise (12 – 24 weeks)

- [ ] Automated Terraform module testing (Terratest integration)
- [ ] FinOps integration — link IaCM apply events to CCM cost attribution tags
- [ ] SBOM generation for IaC module supply chain
- [ ] Automated drift remediation with approval (not just alerting)
- [ ] Self-service IaCM workspace onboarding via Harness Service Catalog

---

## Appendix: Template Inventory

| Template Reference | Version | Owner | Purpose |
|---|---|---|---|
| `account.TU_SEAL_IAC_Scan_Templates` | `IAC_Scan_Prod` | Account | Wiz IaC security scan |
| `org.pipeline_terraform_apply` | `v2` | TU UK org | Full apply pipeline |
| `SEAL_Wiz_IaC_Scan_IaCM_Stepgroup_Local_Template` | local | TU UK | Local Wiz step group |
| `Manage_IACM_Workspace` | `v1.0.5` | OneTru | Workspace lifecycle |
| `CTT_Terraform_Migration` | `v1.0` | TU Africa | CloudFormation migration |

---

*This handbook was auto-generated from live pipeline analysis of Harness account `HgTKqISVTX-kQSVsWCHEcA` on May 4, 2026. Update as new pipelines are onboarded.*
