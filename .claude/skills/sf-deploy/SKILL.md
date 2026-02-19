---
name: sf-deploy
description: "Salesforce deployments, metadata transfers, sandbox management and security audits. Use this skill when the user wants to deploy between orgs, validate metadata, refresh or clone a sandbox, check who has access to what, review login history or run a compliance/security analysis. Do not use for writing code (use /build) or data imports (use /data)."
user-invokable: true
disable-model-invocation: false
---

# sf-deploy — Salesforce Deploy & Audit

You orchestrate deployments following the team pipeline and perform security/compliance analysis.
You are the gatekeeper between development and production.
Production deploys are NEVER done by Claude — team lead only.

## Org Context

- **DEV_SANDBOX** — Dev Sandbox (source for team deploys)
- **UAT_SANDBOX** — UAT Sandbox (Team Lead validates here)
- **Production** — Team Lead ONLY — never deploy here
- **Pipeline:** DEV_SANDBOX → UAT_SANDBOX → Production
- **API Version:** 62.0 | **Org Type:** Sales Cloud + Service Cloud

### Security Rules
- Always specify org alias explicitly — never trust defaults
- Validate before deploy — always run `--dry-run` / `--test-level` first
- Production: inform team lead, never execute directly
- For destructive changes: ALWAYS ask for confirmation

---

## 0. Pre-Deployment: Check the Manifest

Before deploying, check if manifest files exist:

```bash
ls .deployments/
```

### Deploy a specific ticket
When a user says "deploy SFS-1234 to staging":
1. Read `.deployments/SFS-1234.md` if it exists
2. Show the component list and deploy order to the user
3. If no manifest exists: ask the user what should be deployed — do not guess
4. Validate first (`--dry-run`), then deploy in the correct order
5. After successful deploy: update Status in the manifest to "Deployed to Staging"

### Deployment overview (for Deployment Manager)
When a user asks "what's ready for production" or "show open deployments":
1. Scan `.deployments/` for Status "Deployed to Staging" or "Ready for Production"
2. Show a summary table: Ticket | Description | Components | Status
3. If deploying multiple tickets: check for conflicts (same component in multiple tickets)
4. Show the full deploy plan and ask for confirmation
5. After successful deploy: update Status to "Done" in each manifest

> ⚠️ The manifest is a guide, not a guarantee. Always verify against the actual org state before production deploys — the org is the real source of truth.

---

## 1. Deployment Workflow

### Step 1: Validate (Dry Run)
```bash
# Validate without deploying
sf project deploy validate --source-dir force-app -o UAT_SANDBOX --test-level RunLocalTests
```

### Step 2: Deploy to UAT_SANDBOX (UAT)
```bash
# Deploy specific metadata
sf project deploy start --metadata "ApexClass:AccountTriggerHandler" -o UAT_SANDBOX

# Deploy entire source
sf project deploy start --source-dir force-app/main/default -o UAT_SANDBOX --test-level RunLocalTests --wait 30
```

### Step 3: Deploy specific components
```bash
# Apex Class + Trigger together
sf project deploy start --metadata "ApexClass:AccountTriggerHandler,ApexTrigger:AccountTrigger" -o UAT_SANDBOX

# LWC Component
sf project deploy start --metadata "LightningComponentBundle:accountCard" -o UAT_SANDBOX

# Flow
sf project deploy start --metadata "Flow:Account_Assignment_Flow" -o UAT_SANDBOX

# Custom Field
sf project deploy start --metadata "CustomField:Account.My_Field__c" -o UAT_SANDBOX
```

### Step 4: Verify Deployment
```bash
# Check last deploy status
sf project deploy report -o UAT_SANDBOX

# Verify Apex class exists
sf data query --query "SELECT Id, Name, Status FROM ApexClass WHERE Name = 'AccountTriggerHandler' LIMIT 1" --use-tooling-api -o UAT_SANDBOX
```

### Deploy from Git Diff (only changed files)
```bash
# See what changed vs main branch
git diff --name-only main HEAD

# Deploy only changed metadata
sf project deploy start --metadata "ApexClass:AccountTriggerHandler,ApexClass:AccountTriggerHandlerTest" -o UAT_SANDBOX
```

---

## 2. Retrieve Metadata

### Retrieve from Org
```bash
# Single component
sf project retrieve start --metadata "ApexClass:AccountTriggerHandler" -o DEV_SANDBOX

# Entire object (fields, layouts, triggers)
sf project retrieve start --metadata "CustomObject:Account" -o DEV_SANDBOX

# All Flows
sf project retrieve start --metadata "Flow" -o DEV_SANDBOX

# Multiple types
sf project retrieve start --metadata "ApexClass,ApexTrigger,CustomObject" -o DEV_SANDBOX
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
sf org create sandbox --name NewDevSandbox --clone DEV_SANDBOX --target-org Production --wait 30
```
> Sandbox creation targets Production — inform team lead before running!

### Refresh Sandbox
```bash
sf org refresh sandbox --name DEV_SANDBOX --target-org Production
```
> Confirm with user: refresh wipes all data and config changes in the sandbox!

### Open Org in Browser
```bash
sf org open -o DEV_SANDBOX
sf org open -o UAT_SANDBOX
```

### Check Org Auth Status
```bash
sf org list auth
sf org display -o DEV_SANDBOX
```

---

## 4. Security Audit

### Who Has Access to an Object/Field
```bash
# Profiles with access to Account
sf data query --query "SELECT Id, Name FROM Profile" -o DEV_SANDBOX

# Permission Sets assigned to users
sf data query --query "SELECT Id, PermissionSet.Name, Assignee.Name FROM PermissionSetAssignment WHERE PermissionSet.IsOwnedByProfile = false LIMIT 100" -o DEV_SANDBOX

# FLS: which fields a profile can see (via Tooling API)
sf data query --query "SELECT SobjectType, Field, PermissionsRead, PermissionsEdit FROM FieldPermissions WHERE ParentId IN (SELECT Id FROM PermissionSet WHERE Name = 'My_Permission_Set') LIMIT 100" --use-tooling-api -o DEV_SANDBOX
```

### Login History
```bash
sf data query --query "SELECT UserId, LoginTime, LoginType, SourceIp, Status FROM LoginHistory ORDER BY LoginTime DESC LIMIT 100" -o DEV_SANDBOX
```

### Failed Login Attempts
```bash
sf data query --query "SELECT UserId, LoginTime, SourceIp, Status FROM LoginHistory WHERE Status != 'Success' ORDER BY LoginTime DESC LIMIT 50" -o DEV_SANDBOX
```

### Active Users Without Login (90 days)
```bash
sf data query --query "SELECT Id, Name, Username, LastLoginDate FROM User WHERE IsActive = true AND LastLoginDate < LAST_N_DAYS:90 LIMIT 100" -o DEV_SANDBOX
```

### Sharing Rules & Org-Wide Defaults
```bash
sf data query --query "SELECT SobjectType, DefaultInternalAccess, DefaultExternalAccess FROM EntityDefinition WHERE IsCustomizable = true LIMIT 50" --use-tooling-api -o DEV_SANDBOX
```

---

## 5. Destructive Changes

### Remove Component from Org
```bash
# Create destructiveChanges.xml
# (always confirm with user before executing!)
sf project deploy start --manifest destructiveChanges.xml -o DEV_SANDBOX --test-level NoTestRun
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
→ Run tests in DEV_SANDBOX first: `sf apex run test --test-level RunLocalTests -o DEV_SANDBOX`. Deploy only after coverage is green.

**`Cannot deploy: component is in use`**
→ A Flow or Process Builder references the component. Deactivate the Flow first, deploy, then reactivate.

**`Insufficient access rights on cross-reference id`**
→ The target org is missing a referenced record (e.g., RecordType, User). Check if dependencies exist in UAT_SANDBOX.

**`Sandbox refresh: all data will be lost`**
→ This is expected — confirm with the user before proceeding. Export any needed data first via `/data`.

**`UNKNOWN_EXCEPTION during deploy`**
→ Often a timeout. Check status with `sf project deploy report -o UAT_SANDBOX`. Re-run if it failed mid-deploy.

**`Cannot retrieve: Metadata type not supported in source format`**
→ Some metadata types (e.g., Profiles) require special handling. Use `--metadata-api-version 62.0` flag or retrieve via package.xml.

**`sf org list auth: No orgs found`**
→ Re-authenticate: `sf org login web --alias DEV_SANDBOX`. Check VPN if org is not reachable.

---

## 5. Org Health Check

Quick checks to get an overview of org health — run these before a deployment or on request.

### Test Coverage Overview
```bash
# Classes below 75% coverage
sf data query --query "SELECT ApexClassOrTrigger.Name, NumLinesCovered, NumLinesUncovered FROM ApexCodeCoverageAggregate WHERE NumLinesUncovered > 0 ORDER BY NumLinesUncovered DESC LIMIT 20" --use-tooling-api -o DEV_SANDBOX
```

### Apex Exceptions (last 7 days)
```bash
sf data query --query "SELECT ExceptionType, Message, StackTrace, CreatedDate FROM ApexLog WHERE LogLength > 0 AND CreatedDate = LAST_N_DAYS:7 ORDER BY CreatedDate DESC LIMIT 20" --use-tooling-api -o DEV_SANDBOX
```

### Open / Stuck Bulk Jobs
```bash
sf data query --query "SELECT Id, Operation, Object, State, NumberRecordsFailed, CreatedDate FROM AsyncApexJob WHERE Status IN ('Queued','Processing','Holding') ORDER BY CreatedDate DESC LIMIT 20" -o DEV_SANDBOX
```

### Failed Scheduled Jobs
```bash
sf data query --query "SELECT Id, ApexClass.Name, Status, NumberOfErrors, NextFireTime FROM CronTrigger WHERE Status != 'COMPLETE' LIMIT 20" -o DEV_SANDBOX
```

> These are read-only checks — no changes are made. Run before any production deploy to confirm org is stable.

---

## Working Instructions

- Always validate before deploying to UAT_SANDBOX (`--dry-run` or `--test-level RunLocalTests`)
- Never deploy directly to Production — tell the team lead and hand off
- Before sandbox refresh: warn user that data will be wiped, ask for confirmation
- For destructive changes: show what will be deleted, ask for confirmation
- Retrieve metadata before overwriting — never deploy blind
- Security audits are read-only — never change permissions without explicit request
- Check `.deployments/` before deploying — if no manifest exists, ask the user what to deploy
- After every successful deployment: update the manifest Status field
- If the same component appears in multiple ticket manifests: flag the conflict before deploying
