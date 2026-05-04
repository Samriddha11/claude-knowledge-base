# Claude Prompt Templates

Ready-to-use prompts for generating Knowledge Base documents via the Harness Agents in Cursor.

---

## BVR Report

```
Generate a Business Value Review for [customer name] for [month year].
Account ID: [harness account id]

Include:
- Executive summary with top 3 value metrics
- Cost breakdown by cloud provider and business unit (last 30 days)
- Budget health status — which budgets are at risk
- Top 5 cost anomalies with root cause
- Top 10 open savings recommendations with monthly savings
- Commitment orchestration coverage and savings
- Month-over-month trend chart

Format as a structured markdown report with YAML frontmatter:
  title: "[Customer] BVR — [Month Year]"
  subtitle: "Business Value Review"
  customer: "[Customer Name]"
  date: "[Month Year]"
  category: bvr
  author: "Harness CCM FinOps Agent"

Save to: docs/bvr/[customer-slug]-bvr-[mmm-yyyy].md
```

---

## FinOps Cost Review

```
Generate a monthly FinOps cost review for [perspective/account] for [month].

Include:
- Total spend with MoM trend
- Top 10 cost drivers by service/project with trend
- Budget health table (all budgets: actual vs budget vs forecast)
- Daily cost timeseries chart (last 30 days)
- Top anomalies detected this month
- Commitment coverage % and savings

Format with frontmatter:
  category: finops
  author: "Harness CCM FinOps Agent"
```

---

## IaCM Pipeline Analysis

```
Analyse all IaCM pipelines in account [account id].

For each pipeline, extract:
- Stage types and step types
- Security scanning tools integrated
- Approval gate usage
- Drift detection
- Template references

Then generate:
1. A per-pipeline summary table
2. Security coverage heatmap
3. Gaps and P1/P2/P3 remediation items
4. Recommended pipeline blueprint

Format as a best practices handbook with frontmatter:
  category: iacm
  author: "Harness IaCM Agent"
```

---

## Well-Architected Assessment

```
Assess the IaCM maturity of account [account id] against the
IaCM Well-Architected Framework (6 pillars).

For each pillar (Pipeline Design, Security, State Management,
Workspace Governance, Cost & FinOps, Developer Experience):
- Score current state 1-5 with evidence
- Identify top 3 gaps
- Recommend 2-3 specific improvement actions

Generate a maturity report with a scorecard table and roadmap.

Format with frontmatter:
  title: "[Account] IaCM Maturity Assessment"
  category: iacm
  author: "Harness IaCM Agent"
```

---

## Anomaly Investigation

```
Investigate the cost spike detected on [date] in [account/perspective].

Follow the standard triage pattern:
1. Show daily cost timeseries for the 2 weeks around the spike
2. Identify which cloud/service drove the spike
3. Drill to project/account level
4. Show SKU/line-item breakdown
5. Determine if one-off or recurring

Generate a triage report with inline charts and root cause summary.

Format with frontmatter:
  title: "Cost Spike Investigation — [Date]"
  category: finops
  author: "Harness CCM FinOps Agent"
```
