# notes-obsidian

Personal study notes vault, primarily focused on **AWS certifications**. Edited in [Obsidian](https://obsidian.md/) and synced to GitHub via the [github-sync](https://github.com/akifozdemir/github-sync) plugin.

Repo: [dmartinochoa/notes-obsidian](https://github.com/dmartinochoa/notes-obsidian)

---

## Current focus

**AWS Certified Security — Specialty (SCS-C03)** → start at [`AWS/Security Specialist/README.md`](AWS/Security%20Specialist/README.md), a self-authored master summary keyed to the six SCS-C03 domains.

---

## Layout

```
notes/
├── AWS/
│   ├── Cloud Practitioner Essentials/   # CLF foundations (M1–M13)
│   ├── DevOps Engineer Professional/    # DOP-C02 prep, organized by exam domain
│   └── Security Specialist/             # SCS-C03 prep — current focus
├── AWS Test projects/                   # hands-on code experiments
└── README.md
```

### [`AWS/Cloud Practitioner Essentials/`](AWS/Cloud%20Practitioner%20Essentials/)
Foundational notes from AWS Cloud Practitioner Essentials, one file per module (M1–M13). Covers compute, networking, storage, databases, security, ML/analytics, pricing, migration, and the Well-Architected Framework.

### [`AWS/DevOps Engineer Professional/`](AWS/DevOps%20Engineer%20Professional/)
DOP-C02 notes broken out by exam domain. See [`README.md`](AWS/DevOps%20Engineer%20Professional/README.md) for the full table of contents. Highlights:

- `01-sdlc-automation/` — Code* suite, CodeGuru, EC2 Image Builder, Amplify, Jenkins
- `02-configuration-management-and-iac/` — CloudFormation, CDK, SAM, OpsWorks, Service Catalog, Step Functions, SSM, AppConfig
- `03-resilient-cloud-solutions/` — Compute, networking, data, DR
- `04-monitoring-and-logging/` — CloudWatch, Athena, OpenSearch, Lookout
- `05-incident-and-event-response/` — CloudTrail, EventBridge, SNS/SQS, X-Ray
- `06-security-and-compliance/` — Organizations, IAM Identity Center, Config, Control Tower, GuardDuty, Inspector, Macie, Detective, WAF
- `07-other-services/` — CloudFront, Redshift, QuickSight, Copilot, A2C, tagging
- `cheatsheets/` — quick-reference tables (CLI commands, limits, comparisons, exam tips)
- `code examples/` — working CloudFormation templates and SAM apps

### [`AWS/Security Specialist/`](AWS/Security%20Specialist/) *(current focus)*
SCS-C03 study material:

- [`README.md`](AWS/Security%20Specialist/README.md) — master summary, six-domain breakdown, cheat sheets
- `Chapter 1 – Security 101.md` … `Chapter 8 – Troubleshooting Scenarios.md` — long-form chapter notes
- `Quiz/` — self-quizzes per chapter
- `docs/domain-1-detection.md` … `docs/domain-6-governance.md` — task-statement-aligned breakdowns with sub-skills and service mappings
- `Practice Exams/` — ACloudGuru practice exam, annotated example questions, official exam guide
- `AWS Whitepaper Summaries/` — DDoS Best Practices, KMS Best Practices, Logging at Scale
- `Cheetsheet.md` — long-form cheatsheet by chapter
- `General Exam Tips.md` — test-taking heuristics
- `NETWORKS GLOSSARY.md` — networking terms

### [`AWS Test projects/`](AWS%20Test%20projects/)
Hands-on code experiments referenced from the notes.

- `lambda-process-SAM/` — SAM-deployed Lambda image resizer (see its own [`README.md`](AWS%20Test%20projects/lambda-process-SAM/README.md))

---

## SCS-C03 quick reference

| Domain | Weight | Folder link |
|---|---|---|
| 1. Detection | 16% | [`docs/domain-1-detection.md`](AWS/Security%20Specialist/docs/domain-1-detection.md) |
| 2. Incident Response | 14% | [`docs/domain-2-incident-response.md`](AWS/Security%20Specialist/docs/domain-2-incident-response.md) |
| 3. Infrastructure Security | 18% | [`docs/domain-3-infrastructure-security.md`](AWS/Security%20Specialist/docs/domain-3-infrastructure-security.md) |
| 4. Identity & Access Management | 20% | [`docs/domain-4-iam.md`](AWS/Security%20Specialist/docs/domain-4-iam.md) |
| 5. Data Protection | 18% | [`docs/domain-5-data-protection.md`](AWS/Security%20Specialist/docs/domain-5-data-protection.md) |
| 6. Security Foundations & Governance | 14% | [`docs/domain-6-governance.md`](AWS/Security%20Specialist/docs/domain-6-governance.md) |

Passing score: 750 / 1000.

---

## Conventions

- Notes are CommonMark Markdown for Obsidian compatibility.
- Filename typos are sometimes preserved (e.g. `Cheetsheet.md`) to avoid breaking existing wiki-links.
- `AWS Test projects/lambda-process-SAM/` may carry `node_modules/` and `.aws-sam/` build output locally — both are gitignored. Recommended Obsidian search exclusions: `node_modules`, `.aws-sam`, `.git`.

## Sync

Push/pull is handled by the `github-sync` Obsidian plugin (configured under `.obsidian/plugins/github-sync/`, gitignored). Standard `git` also works from the repo root.

## License

Personal study notes — no license; not intended for redistribution. Some content paraphrases AWS documentation and third-party training material (A Cloud Guru, etc.) for personal reference.
