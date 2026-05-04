# IaCM Pipeline Patterns Reference

> Four production-grade patterns observed from live Harness IaCM account analysis.

## Pattern 1 — Unified Lifecycle Pipeline

**Best for:** Teams managing many workspaces who want minimal pipeline sprawl.

A single pipeline controls all lifecycle actions via a `TF_Action` stage variable.

```
init → plan → [Security Scan] → drift_detect → PARALLEL
                                                  ├─► apply   (TF_Action == 'apply')
                                                  └─► destroy (TF_Action == 'destroy')
```

**Observed in:** TU Africa / AISADMINIAC (AWS infrastructure)

---

## Pattern 2 — Dedicated Per-Action Pipelines

**Best for:** PR-driven GitOps workflows with webhook triggers.

Separate pipelines for plan, apply, destroy, drift — all backed by a shared org-level template.

```
pipeline_plan    → plan only
pipeline_apply   → [Wiz Scan] → IACMApproval → apply
pipeline_destroy → [Wiz Scan] → IACMApproval → destroy
pipeline_drift   → detect-drift
```

**Observed in:** TU UK (The First Project, ArronTest, TF Automation Testing)

---

## Pattern 3 — Multi-Stage Orchestration

**Best for:** Complex multi-environment deployments with dynamic credential injection.

```
Stage 1: Pre Setup     → Vault AppRole login, env-routed (dev/stg/prod)
Stage 2: Workspace Mgmt → Dynamic workspace creation via template
Stage 3: IaCM Execution → Plan, scan, approve, apply
```

**Observed in:** OneTru / Vanguard (GCP/GKE)

---

## Pattern 4 — Migration Pipeline

**Best for:** Bringing existing CloudFormation/manual infrastructure under Terraform management.

```
terraform import → [Wiz IaC Scan] → terraform plan (expect zero diff) → approval → commit
```

**Observed in:** TU Africa / AISADMINIAC (`Cloudformation_to_Terraform` pipeline)
