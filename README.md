# sf_cc_setup — Salesforce Claude Code Skills

A ready-to-use Claude Code setup for Salesforce teams.
Clone this repo into your Salesforce project and get 3 specialized AI skills out of the box.

## What's included

| Skill | Invoke | Purpose |
|-------|--------|---------|
| `sf-build` | `/build` | Apex, LWC, Flows, Validation Rules, Users, Config |
| `sf-data` | `/data` | SOQL, CSV imports, SFDMU, Data Quality |
| `sf-deploy` | `/deploy` | Deployments, Sandbox management, Security Audit |

**Simple mental model:**
- *"I need to build or change something"* → `/build`
- *"I need to view or move data"* → `/data`
- *"I need to deploy or audit something"* → `/deploy`

## Setup

### 1. Copy into your Salesforce project root
```bash
# Option A: clone directly into your project
git clone https://github.com/NicoIO-beep/sf_cc_setup.git .

# Option B: copy into an existing project
cp -r sf_cc_setup/CLAUDE.md sf_cc_setup/.claude your-project/
```

### 2. Replace the org aliases

Open `CLAUDE.md` and the 3 skill files and replace the sandbox aliases with your own:

| Placeholder | Replace with |
|-------------|-------------|
| `DEV_SANDBOX` | Your dev sandbox alias |
| `UAT_SANDBOX` | Your UAT sandbox alias |

```bash
# Quick replace (macOS/Linux)
grep -rl "DEV_SANDBOX" . | xargs sed -i 's/DEV_SANDBOX/YOUR_DEV_ALIAS/g'
grep -rl "UAT_SANDBOX" . | xargs sed -i 's/UAT_SANDBOX/YOUR_UAT_ALIAS/g'
```

### 3. Start Claude Code and use the skills
```bash
claude
```

Then type a skill command followed by your task:
```
/build   Write me an Apex trigger for Account
/data    SOQL query for all open Opportunities this quarter
/deploy  Deploy AccountTrigger from dev to UAT
```

## Security notes

- No credentials, tokens or passwords are stored in this repo
- Org aliases are generic placeholders — replace them with your own
- All skills follow the principle: never deploy to Production directly
- PII and real customer data must never be used in prompts

## Requirements

- [Claude Code](https://claude.ai/claude-code)
- [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`)
- Authenticated org connections (`sf org login web --alias YOUR_ALIAS`)

**New to this setup? No Salesforce DX project yet? On Windows without admin rights?**
→ Read [SETUP.md](SETUP.md) for step-by-step instructions.
