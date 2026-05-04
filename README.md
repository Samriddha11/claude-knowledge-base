# Claude AI Knowledge Base

> Internal Harness IDP portal for all Claude AI-generated documents, frameworks, and reports.

## Structure

```
claude-knowledge-base/
├── catalog-info.yaml          IDP catalog entities (System + Components)
├── mkdocs.yml                 TechDocs navigation and theme config
├── docs/
│   ├── index.md               Portal home page
│   ├── bvr/                   Business Value Reviews
│   ├── iacm/                  IaCM frameworks and analysis
│   ├── finops/                FinOps and CCM reports
│   ├── best-practices/        Engineering best practices
│   ├── how-to-contribute.md   Guide for adding documents
│   └── prompt-templates.md    Claude prompts for each doc type
└── .harness/
    └── publish-artifact.yaml  Pipeline to auto-publish Claude artifacts
```

## Setup in Harness IDP

### 1. Create GitHub repo
Create `Samriddha11/claude-knowledge-base` and push this directory.

### 2. Register in IDP
Go to **Harness IDP → Catalog → Register Component** and enter:
```
https://github.com/Samriddha11/claude-knowledge-base/blob/main/catalog-info.yaml
```

### 3. Enable TechDocs
In IDP settings, ensure TechDocs builder is configured (basic or CI/CD mode).

### 4. Browse
Navigate to **IDP → Catalog → claude-knowledge-base** to see the portal.

## Adding Documents

See [docs/how-to-contribute.md](docs/how-to-contribute.md) or use the
`publish-claude-artifact` pipeline in Harness.
