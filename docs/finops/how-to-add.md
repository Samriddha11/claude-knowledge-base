# How to Add FinOps Reports

## Step 1 — Generate the Report

Use the Harness CCM FinOps Agent:

```
Generate a [report type] for [perspective/account] covering [period].
Save the markdown to:
docs/finops/[account]-[report-type]-[period].md
```

## Step 2 — Required Frontmatter

```yaml
---
title: "[Account] [Report Type] — [Period]"
subtitle: "[Cloud Provider] Cost Analysis"
customer: "[Account Name]"
date: "[Month Year]"
category: finops
author: "Harness CCM FinOps Agent"
---
```

## Step 3 — Update Navigation

Add to `mkdocs.yml`:

```yaml
nav:
  - FinOps & CCM Reports:
    - Overview: finops/index.md
    - "[Account] [Period]": finops/[account]-[type]-[period].md
```

## Step 4 — PR and Publish

```bash
git checkout -b finops/[account]-[period]
git add docs/finops/ mkdocs.yml
git commit -m "docs(finops): add [account] [report type] [period]"
git push && # open PR
```
