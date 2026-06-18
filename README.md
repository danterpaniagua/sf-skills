# sf-skills

Skill library for Claude Code agents. Skills are `.md` files at the repo root, discoverable as slash commands when this repo is mounted as a submodule at `.claude/commands/`.

## Usage

```bash
git submodule add git@github.com:danterpaniagua/sf-skills.git .claude/commands
```

## Skills

### `loyalty-*` — SmartLoyalty SQL Server

| Skill | Invocation | Scope |
|---|---|---|
| `loyalty-fraud-points` | `/loyalty-fraud-points` | Transfer fraud, accumulation anomaly, manual assignment exploit |
| `loyalty-fraud-pos` | `/loyalty-fraud-pos` | POS terminal manipulation, franchise network fraud |
| `loyalty-dba-investigation` | `/loyalty-dba-investigation` | DBA investigation — PNSSRL |
| `loyalty-sre-output` | `/loyalty-sre-output` | Formatted outputs for PM, IT, and Jira |
| `loyalty-azure-nsg` | `/loyalty-azure-nsg` | Azure NSG audit |

### `sp-*` — SmartPedidos (Node.js/Express)

| Skill | Invocation | Scope |
|---|---|---|
| `sp-log-improvements` | `/sp-log-improvements` | Apply logging standard to a service codebase |
| `sp-srp-refactor` | `/sp-srp-refactor` | SRP violation analysis and Jira story generation |
| `sp-static-analysis` | `/sp-static-analysis` | Static analysis for critical defects and vulnerabilities |
| `sp-tech-debt` | `/sp-tech-debt` | Record technical debt items to central log |
| `sp-sre-output` | `/sp-sre-output` | Formatted outputs for PM, IT, and Jira |
| `ops-aws` | `/ops-aws` | AWS / SQS triage — platforms-service and concentrador-service |

### `itiano-*` — Itiano Django Project

| Skill | Invocation | Scope |
|---|---|---|
| `itiano-dba-postgres` | `/itiano-dba-postgres` | PostgreSQL administration and diagnostics |
| `itiano-django-observability` | `/itiano-django-observability` | Add operational logging to Django/PostgreSQL code |
| `itiano-scope-driven-development` | `/itiano-scope-driven-development` | Requirements analysis and scope management |
| `itiano-scope-validation` | `/itiano-scope-validation` | Validate implementation against approved scope |
| `itiano-test-planning` | `/itiano-test-planning` | Create validation and testing plans |

### `ope-*` — Operations (Infrastructure)

| Skill | Invocation | Scope |
|---|---|---|
| `ope-azure` | `/ope-azure` | Azure AD DS health/alerts, Kerberos policy, VMs, NSGs, Monitor |
| `ope-aws` | `/ope-aws` | EC2, SQS, CloudWatch, IAM review, ECS, Fargate, ALB/NLB |
| `ope-sre-output` | `/ope-sre-output` | Event artifacts: Jira tickets, closure reports, emails, ops-events log |

### Cross-project

| Skill | Invocation | Scope |
|---|---|---|
| `doc-audit` | `/doc-audit` | Documentation and context integrity audit |

## Structure

Skills are prefixed by product to avoid naming collisions when multiple product skill sets share the same `.claude/commands/` directory. Source of truth for `loyalty-*` skills is `loyalty/.claude/commands/` — sf-skills is synced from it.
