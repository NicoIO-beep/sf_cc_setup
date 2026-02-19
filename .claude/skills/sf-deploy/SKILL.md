---
name: sf-deploy
description: "Salesforce deployments, metadata transfers, sandbox management and security audits. Use this skill when the user wants to deploy between orgs, validate metadata, refresh or clone a sandbox, check who has access to what, review login history or run a compliance/security analysis. Do not use for writing code (use /build) or data imports (use /data)."
user-invokable: true
disable-model-invocation: true
---

# sf-deploy — Salesforce Deploy & Audit

You orchestrate deployments following the team pipeline and perform security/compliance analysis.
You are the gatekeeper between development and production.
Production deploys are NEVER done by Claude — team lead only.

## Org Context

- **ClaudeTest** — Dev Sandbox (source for team deploys)
- **nicosb1** — UAT Sandbox (Team Lead validates here)
- **Production** — Team Lead ONLY — never deploy here
- **Pipeline:** ClaudeTest → nicosb1 → Production
- **API Version:** 62.0 | **Org Type:** Sales Cloud + Service Cloud

### Security Rules
- Always specify org alias explicitly — never trust defaults
- Validate before deploy — always run `--dry-run` / `--test-level` first
- Production: inform team lead, never execute directly
- For destructive changes: ALWAYS ask for confirmation

---

## 1. Deployment Workflow

### Step 1: Validate (Dry Run)
```bash
# Validate without deploying
sf project deploy validate --source-dir force-app -o nicosb1 --test-level RunLocalTests
```

### Step 2: Deploy to nicosb1 (UAT)
```bash
# Deploy specific metadata
sf project deploy start --metadata "ApexClass:AccountTriggerHandler" -o nicosb1

# Deploy entire source
sf project deploy start --source-dir force-app/main/default -o nicosb1 --test-level RunLocalTests --wait 30
```

### Step 3: Deploy specific components
```bash
# Apex Class + Trigger together
sf project deploy start --metadata "ApexClass:AccountTriggerHandler,ApexTrigger:AccountTrigger" -o nicosb1

# LWC Component
sf project deploy start --metadata "LightningComponentBundle:accountCard" -o nicosb1

# Flow
sf project deploy start --metadata "Flow:Account_Assignment_Flow" -o nicosb1

# Custom Field
sf project deploy start --metadata "CustomField:Account.My_Field__c" -o nicosb1
```

### Step 4: Verify Deployment
```bash
# Check last deploy status
sf project deploy report -o nicosb1

# Verify Apex class exists
sf data query --query "SELECT Id, Name, Status FROM ApexClass WHERE Name = 'AccountTriggerHandler' LIMIT 1" --use-tooling-api -o nicosb1
```

### Deploy from Git Diff (only changed files)
```bash
# See what changed vs main branch
git diff --name-only main HEAD

# Deploy only changed metadata
sf project deploy start --metadata "ApexClass:AccountTriggerHandler,ApexClass:AccountTriggerHandlerTest" -o nicosb1
```

---

## 2. Retrieve Metadata

### Retrieve from Org
```bash
# Single component
sf project retrieve start --metadata "ApexClass:AccountTriggerHandler" -o ClaudeTest

# Entire object (fields, layouts, triggers)
sf project retrieve start --metadata "CustomObject:Account" -o ClaudeTest

# All Flows
sf project retrieve start --metadata "Flow" -o ClaudeTest

# Multiple types
sf project retrieve start --metadata "ApexClass,ApexTrigger,CustomObject" -o ClaudeTest
```

### Generate Package.xml for Deployment
```bash
sf project generate manifest --source-dir force-app/main/default --name package
```

---

## 3. Sandbox Management

### List All Sandboxes
```bash
sf org list
```

### Create Sandbox
```bash
sf org create sandbox --name NewDevSandbox --clone ClaudeTest --target-org Production --wait 30
```
> Sandbox creation targets Production — inform team lead before running!

### Refresh Sandbox
```bash
sf org refresh sandbox --name ClaudeTest --target-org Production
```
> Confirm with user: refresh wipes all data and config changes in the sandbox!

### Open Org in Browser
```bash
sf org open -o ClaudeTest
sf org open -o nicosb1
```

### Check Org Auth Status
```bash
sf org list auth
sf org display -o ClaudeTest
```

---

## 4. Security Audit

### Who Has Access to an Object/Field
```bash
# Profiles with access to Account
sf data query --query "SELECT Id, Name FROM Profile" -o ClaudeTest

# Permission Sets assigned to users
sf data query --query "SELECT Id, PermissionSet.Name, Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.IsOwnedByProfile = false LIMIT 100" -o ClaudeTest

# FLS: which fields a profile can see (via Tooling API)
sf data query --query "SELECT SobjectType, Field, PermissionsRead, PermissionsEdit FROM FieldPermissions WHERE ParentId IN (SELECT Id FROM PermissionSet WHERE Name = 'My_Permission_Set') LIMIT 100" --use-tooling-api -o ClaudeTest
```

### Login History
```bash
sf data query --query "SELECT UserId, LoginTime, LoginType, SourceIp, Status FROM LoginHistory ORDER BY LoginTime DESC LIMIT 100" -o ClaudeTest
```

### Failed Login Attempts
```bash
sf data query --query "SELECT UserId, LoginTime, SourceIp, Status FROM LoginHistory WHERE Status != 'Success' ORDER BY LoginTime DESC LIMIT 50" -o ClaudeTest
```

### Active Users Without Login (90 days)
```bash
sf data query --query "SELECT Id, Name, Username, LastLoginDate FROM User WHERE IsActive = true AND LastLoginDate < LAST_N_DAYS:90 LIMIT 100" -o ClaudeTest
```

### Sharing Rules & Org-Wide Defaults
```bash
sf data query --query "SELECT SobjectType, DefaultInternalAccess, DefaultExternalAccess FROM EntityDefinition WHERE IsCustomizable = true LIMIT 50" --use-tooling-api -o ClaudeTest
```

---

## 5. Destructive Changes

### Remove Component from Org
```bash
# Create destructiveChanges.xml
# (always confirm with user before executing!)
sf project deploy start --manifest destructiveChanges.xml -o ClaudeTest --test-level NoTestRun
```

**destructiveChanges.xml format:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>OldClassName</members>
        <name>ApexClass</name>
    </types>
    <version>62.0</version>
</Package>
```

---

## 6. Common Errors & Solutions

**`Deploy failed: Test coverage is 0%`**
→ Run tests in ClaudeTest first: `sf apex run test --test-level RunLocalTests -o ClaudeTest`. Deploy only after coverage is green.

**`Cannot deploy: component is in use`**
→ A Flow or Process Builder references the component. Deactivate the Flow first, deploy, then reactivate.

**`Insufficient access rights on cross-reference id`**
→ The target org is missing a referenced record (e.g., RecordType, User). Check if dependencies exist in nicosb1.

**`Sandbox refresh: all data will be lost`**
→ This is expected — confirm with the user before proceeding. Export any needed data first via `/data`.

**`UNKNOWN_EXCEPTION during deploy`**
→ Often a timeout. Check status with `sf project deploy report -o nicosb1`. Re-run if it failed mid-deploy.

**`Cannot retrieve: Metadata type not supported in source format`**
→ Some metadata types (e.g., Profiles) require special handling. Use `--metadata-api-version 62.0` flag or retrieve via package.xml.

**`sf org list auth: No orgs found`**
→ Re-authenticate: `sf org login web --alias ClaudeTest`. Check VPN if org is not reachable.

---

## Working Instructions

- Always validate before deploying to nicosb1 (`--dry-run` or `--test-level RunLocalTests`)
- Never deploy directly to Production — tell the team lead and hand off
- Before sandbox refresh: warn user that data will be wiped, ask for confirmation
- For destructive changes: show what will be deleted, ask for confirmation
- Retrieve metadata before overwriting — never deploy blind
- Security audits are read-only — never change permissions without explicit request
