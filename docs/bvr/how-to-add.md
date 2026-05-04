# How to Add a BVR

Follow these steps to publish a Claude-generated BVR to the Knowledge Base portal.

## Step 1 — Generate the BVR

Use the Harness CCM FinOps Agent in Cursor:

```
Generate a monthly BVR for [account name] for [month/year].
Save as markdown with YAML frontmatter at:
/path/to/claude-knowledge-base/docs/bvr/[account]-[month]-[year].md
```

The agent will:
- Pull live cost data, budgets, anomalies, recommendations
- Generate a formatted markdown report
- Save it to the correct location

## Step 2 — Add YAML Frontmatter

Every BVR document must start with this frontmatter:

```yaml
---
title: "[Account Name] BVR — [Month Year]"
subtitle: "Business Value Review"
customer: "[Account Name]"
date: "[Month Year]"
category: bvr
author: "Harness CCM Agent"
---
```

## Step 3 — Update Navigation

Add the new BVR to `mkdocs.yml` under the BVR section:

```yaml
nav:
  - Business Value Reviews:
      - Overview: bvr/index.md
      - How to Add a BVR: bvr/how-to-add.md
      - "[Account] [Month] [Year]": bvr/[account]-[month]-[year].md  # ← add here
```

## Step 4 — Open a PR

```bash
git checkout -b bvr/[account]-[month]-[year]
git add docs/bvr/ mkdocs.yml
git commit -m "docs(bvr): add [account] BVR for [month year]"
git push origin bvr/[account]-[month]-[year]
# open PR → merge → TechDocs rebuilds automatically
```

## Step 5 — Verify in IDP

After the PR merges, visit the Harness IDP TechDocs page for this portal. The new BVR will appear in the left navigation under **Business Value Reviews**.
