# sf-skills

Skill library for Claude Code agents. Skills are `.md` files at the repo root, discoverable as slash commands when this repo is mounted as a submodule at `.claude/commands/`.

## Usage

```bash
git submodule add git@github.com:danterpaniagua/sf-skills.git .claude/commands
```

## Skills

| Skill | Invocation | Scope |
|---|---|---|
| `loyalty-fraud-points` | `/loyalty-fraud-points` | Transfer fraud, accumulation anomaly, manual assignment exploit — SmartLoyalty |
| `loyalty-fraud-pos` | `/loyalty-fraud-pos` | POS terminal manipulation, franchise network fraud — SmartLoyalty |
| `loyalty-dba-investigation` | `/loyalty-dba-investigation` | DBA investigation — PNSSRL |
| `loyalty-sre-output` | `/loyalty-sre-output` | Formatted outputs for PM, IT, and Jira — SmartLoyalty |
| `loyalty-azure-nsg` | `/loyalty-azure-nsg` | Azure NSG audit |
| `doc-audit` | `/doc-audit` | Documentation and context integrity audit |

## Structure

Skills are prefixed by product (`loyalty-*`) to avoid naming collisions when multiple product skill sets share the same `.claude/commands/` directory.
