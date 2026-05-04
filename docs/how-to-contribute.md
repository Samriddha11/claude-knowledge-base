# Contributing Claude Artifacts to the Knowledge Base

This guide explains how to add any Claude AI-generated document to the portal.

---

## The Three-Step Flow

```
1. GENERATE          2. COMMIT            3. PUBLISH
─────────────        ──────────────       ──────────────────
Claude / Harness  →  PR to this repo  →  IDP TechDocs auto-
Agent produces       docs/ subfolder     rebuilds on merge
markdown report
```

---

## Folder Structure

```
docs/
├── bvr/              ← Business Value Reviews
│   └── [account]-[month]-[year].md
├── iacm/             ← IaCM frameworks and analysis
│   └── [topic].md
├── finops/           ← FinOps and CCM reports
│   └── [account]-[type]-[period].md
└── best-practices/   ← Engineering standards
    └── [topic].md
```

---

## Required Frontmatter

Every document **must** start with YAML frontmatter:

```yaml
---
title: "Document Title"
subtitle: "Optional subtitle"
customer: "Account or Team Name"
date: "Month Year"
category: bvr | iacm | finops | best-practices
author: "Harness CCM Agent | IaCM Agent | Your Name"
tags:
  - relevant-tag
  - another-tag
---
```

---

## Naming Conventions

| Category | File naming pattern | Example |
|---|---|---|
| BVR | `[account]-bvr-[mmm-yyyy].md` | `acme-bvr-may-2026.md` |
| FinOps report | `[account]-[type]-[mmm-yyyy].md` | `transunion-cost-review-may-2026.md` |
| IaCM doc | `[topic-slug].md` | `well-architected-framework.md` |
| Best practice | `[topic-slug].md` | `terraform-module-standards.md` |

---

## Adding to Navigation

Edit `mkdocs.yml` and add your file to the correct `nav` section:

```yaml
nav:
  - Business Value Reviews:
      - Overview: bvr/index.md
      - "ACME BVR May 2026": bvr/acme-bvr-may-2026.md   # ← add here
```

---

## Automated Publishing via Pipeline

The repo includes a Harness pipeline (`.harness/publish-artifact.yaml`) that automates this entire flow. To use it:

1. Generate your report with the Harness Agent
2. Trigger the `publish-claude-artifact` pipeline with:
   - `artifact_path`: local path to the markdown file
   - `target_folder`: `bvr` | `iacm` | `finops` | `best-practices`
   - `document_title`: human-readable title
3. The pipeline opens a PR automatically

---

## Claude Prompt Templates

See [Claude Prompt Templates](prompt-templates.md) for ready-to-use prompts that generate correctly-formatted documents for each category.
