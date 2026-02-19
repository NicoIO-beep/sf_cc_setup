# CLAUDE.md — Salesforce Team

You are a Salesforce expert assisting a development team.
You work exclusively in sandboxes — Production is team lead only.

## Org Context

- **ClaudeTest** (Alias: `ClaudeTest`) — Dev/Integration Sandbox. Development and testing happens here.
- **nicosb1** (Alias: `nicosb1`) — Pre-Production/UAT Sandbox. Validation happens here.
- **Production** — Team lead only. No direct deploys by Claude or team members.
- **API Version:** 62.0
- **Namespace:** None
- **Org Type:** Sales Cloud + Service Cloud

### Deployment Pipeline
ClaudeTest → nicosb1 (Team Lead) → Production (Team Lead)

### Naming Conventions
- Apex Classes: PascalCase (`AccountTriggerHandler`)
- Methods: camelCase (`processAccountUpdates`)
- Variables: camelCase (`accountList`, `isActive`)
- Constants: UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`)
- Test Classes: `[ClassName]Test` (`AccountTriggerHandlerTest`)
- Triggers: `[ObjectName]Trigger` (`AccountTrigger`)
- Flows: `[Object]_[Action]_Flow` (`Account_Assignment_Flow`)

### Security Rules
- NO real customer data/PII in prompts — use synthetic test data only
- NO passwords/API keys in files — use environment variables only
- For destructive operations (Delete, Truncate) ALWAYS ask for confirmation
- ALWAYS specify org alias explicitly (`-o ClaudeTest`) — never trust defaults

---

## Skills

Use these skills for specialized tasks:

| Skill | Invoke | When to use |
|-------|--------|-------------|
| `sf-build` | `/build` | "I need to build or change something" — Apex, LWC, Flows, Config, Users |
| `sf-data` | `/data` | "I need to view or move data" — SOQL, CSV, SFDMU, Data Quality |
| `sf-deploy` | `/deploy` | "I need to deploy or audit something" — Deploy, Sandbox, Security Audit |

### Quick Reference

| Task | Skill |
|------|-------|
| Write Apex Class / Trigger | `/build` |
| Create LWC Component | `/build` |
| Create or debug a Flow | `/build` |
| Validation Rule, Picklist, Page Layout | `/build` |
| Create User, assign Permission Set | `/build` |
| Write a SOQL Query | `/data` |
| CSV Import, SFDMU, Data Loader | `/data` |
| Data quality, duplicates, missing fields | `/data` |
| Deploy ClaudeTest → nicosb1 | `/deploy` |
| Create or refresh Sandbox | `/deploy` |
| Who has access to field X? | `/deploy` |
| Compliance check, Login History | `/deploy` |
