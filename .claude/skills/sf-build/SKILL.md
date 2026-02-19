---
name: sf-build
description: "Salesforce build and configure: Apex classes, triggers, LWC, Flows, validation rules, picklists, page layouts, user management and permission sets. Use this skill when the user wants to build, create, write or change anything in Salesforce — code or declarative config. Also when debugging Apex, fixing a Flow, or setting up users. Do not use for data imports (use /data) or deployments (use /deploy)."
user-invokable: true
disable-model-invocation: true
---

# sf-build — Salesforce Build & Configure

You are the Salesforce developer and administrator for this team.
You write Apex code following team standards, build LWC components, configure Flows
and handle all declarative setup. You work in DEV_SANDBOX — deployments go via /deploy.

## Org Context

- **DEV_SANDBOX** — Dev Sandbox (your workspace)
- **UAT_SANDBOX** — UAT Sandbox (Team Lead validates here)
- **Production** — Team Lead only — never deploy directly
- **API Version:** 62.0 | **Org Type:** Sales Cloud + Service Cloud

### Naming Conventions
- Apex Classes: PascalCase — `AccountTriggerHandler`
- Test Classes: `[ClassName]Test` — `AccountTriggerHandlerTest`
- Triggers: `[ObjectName]Trigger` — `AccountTrigger`
- Flows: `[Object]_[Action]_Flow` — `Account_Assignment_Flow`
- Methods: camelCase | Variables: camelCase | Constants: UPPER_SNAKE_CASE

### Security Rules
- Always use `-o DEV_SANDBOX` — never trust defaults
- No PII/real customer data — synthetic test data only
- Ask for confirmation before any destructive operation

---

## 1. Apex Development

### Create Apex Class
```bash
sf apex generate class --name AccountTriggerHandler --output-dir force-app/main/default/classes -o DEV_SANDBOX
```

**Handler Pattern (team standard):**
```apex
public with sharing class AccountTriggerHandler {
    public static void handleBeforeInsert(List<Account> newList) {
        for (Account acc : newList) {
            // logic here
        }
    }

    public static void handleAfterUpdate(List<Account> newList, Map<Id, Account> oldMap) {
        List<Account> changed = new List<Account>();
        for (Account acc : newList) {
            if (acc.Name != oldMap.get(acc.Id).Name) {
                changed.add(acc);
            }
        }
        if (!changed.isEmpty()) processAccountUpdates(changed);
    }

    private static void processAccountUpdates(List<Account> accounts) {
        // bulk-safe logic
    }
}
```

### Create Trigger (one per object — team standard)
```bash
sf apex generate trigger --name AccountTrigger --sobject Account --output-dir force-app/main/default/triggers -o DEV_SANDBOX
```

```apex
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    if (Trigger.isBefore && Trigger.isInsert) AccountTriggerHandler.handleBeforeInsert(Trigger.new);
    if (Trigger.isBefore && Trigger.isUpdate) AccountTriggerHandler.handleBeforeUpdate(Trigger.new, Trigger.oldMap);
    if (Trigger.isAfter && Trigger.isInsert)  AccountTriggerHandler.handleAfterInsert(Trigger.new);
    if (Trigger.isAfter && Trigger.isUpdate)  AccountTriggerHandler.handleAfterUpdate(Trigger.new, Trigger.oldMap);
}
```

### Run Apex Anonymous
```bash
sf apex run --file scripts/apex/myScript.apex -o DEV_SANDBOX
```

### Execute & See Logs
```bash
sf apex run --file scripts/apex/myScript.apex -o DEV_SANDBOX --log-level DEBUG
```

---

## 2. Unit Tests

### Create Test Class
```apex
@IsTest
private class AccountTriggerHandlerTest {
    @TestSetup
    static void makeData() {
        Account acc = new Account(Name = 'Test Account', Industry = 'Technology');
        insert acc;
    }

    @IsTest
    static void testHandleBeforeInsert() {
        Test.startTest();
        Account acc = new Account(Name = 'New Account');
        insert acc;
        Test.stopTest();

        Account result = [SELECT Id, Name FROM Account WHERE Name = 'New Account' LIMIT 1];
        Assert.areEqual('New Account', result.Name);
    }
}
```

### Run Tests
```bash
# Single class
sf apex run test --class-names AccountTriggerHandlerTest -o DEV_SANDBOX --result-format human

# All tests
sf apex run test --test-level RunAllTestsInOrg -o DEV_SANDBOX --result-format human

# With coverage
sf apex run test --class-names AccountTriggerHandlerTest -o DEV_SANDBOX --code-coverage --result-format human
```

**Target: >75% coverage per class**

---

## 3. Lightning Web Components

### Generate LWC
```bash
sf lightning generate component --name accountCard --type lwc --output-dir force-app/main/default/lwc -o DEV_SANDBOX
```

**Basic Component Structure:**
```javascript
// accountCard.js
import { LightningElement, api, wire } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';

export default class AccountCard extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: [NAME_FIELD] })
    account;

    get accountName() {
        return this.account?.data?.fields?.Name?.value ?? '';
    }
}
```

### Deploy LWC to Org
```bash
sf project deploy start --source-dir force-app/main/default/lwc/accountCard -o DEV_SANDBOX
```

---

## 4. Admin Configuration via CLI

### User Management
```bash
# List users
sf data query --query "SELECT Id, Name, Username, IsActive FROM User LIMIT 50" -o DEV_SANDBOX

# Deactivate user (confirm first!)
sf data update record --sobject User --record-id [USER_ID] --values "IsActive=false" -o DEV_SANDBOX

# Assign Permission Set
sf org assign permset --name My_Permission_Set --on-behalf-of user@example.com -o DEV_SANDBOX
```

### Retrieve Metadata for Editing
```bash
# Retrieve custom field
sf project retrieve start --metadata "CustomField:Account.My_Field__c" -o DEV_SANDBOX

# Retrieve picklist (custom object)
sf project retrieve start --metadata "CustomObject:My_Object__c" -o DEV_SANDBOX

# Retrieve Flow
sf project retrieve start --metadata "Flow:Account_Assignment_Flow" -o DEV_SANDBOX
```

### Push Changes Back
```bash
sf project deploy start --source-dir force-app/main/default/objects/Account -o DEV_SANDBOX
```

---

## 5. Common Errors & Solutions

**`FIELD_CUSTOM_VALIDATION_EXCEPTION`**
→ A validation rule is blocking the DML. Find it via Tooling API:
```bash
sf data query --query "SELECT Id, Active, Description FROM ValidationRule WHERE EntityDefinition.QualifiedApiName = 'Account'" --use-tooling-api -o DEV_SANDBOX
```

**`System.LimitException: Too many SOQL queries: 101`**
→ SOQL inside a loop. Move the query outside the loop, collect IDs first, then query once with `WHERE Id IN :idSet`.

**`MIXED_DML_OPERATION`**
→ Setup and non-setup objects in same transaction. Use `System.runAs()` in tests or a future method for setup objects.

**`Cannot deploy: component is in use by a Flow/Process`**
→ Deactivate the Flow/Process Builder first, then deploy, then reactivate.

**`Compile error: Variable does not exist`**
→ Check API names (case-sensitive). Use `sf schema describe sobject --sobject Account -o DEV_SANDBOX` to verify field names.

**`Test coverage: 0%` after deploy**
→ Run `sf apex run test --test-level RunLocalTests -o DEV_SANDBOX` to regenerate coverage data.

**`LWC: [Cannot read properties of undefined]`**
→ Add null-check: use optional chaining `?.` and nullish coalescing `??` before accessing nested properties.

---

## Working Instructions

- One trigger per object — always use the handler pattern
- No SOQL or DML inside loops — always bulkify
- Always specify `-o DEV_SANDBOX` — never trust defaults
- Test classes: use `@TestSetup`, assert with `Assert.areEqual()` (not `System.assertEquals`)
- Before deactivating users or deleting records: ask for confirmation
- Retrieve metadata before editing locally — never edit blind
- After every org-changing operation: update `.deployments/[TICKET-NR].md`
- If no ticket number was provided: ask for one before proceeding with org changes

---

## Windows Compatibility Notes

Most team members are on **Windows without admin rights**. Keep this in mind:

- **SOQL with special characters** (e.g. `|`, `>`) can break in PowerShell. Write them to an Apex Anonymous file and run with `sf apex run --file`:
  ```bash
  # Instead of inline SOQL in PowerShell, use a script file:
  sf apex run --file scripts/apex/myQuery.apex -o DEV_SANDBOX
  ```
- **CSV line endings**: Windows saves files with CRLF (`\r\n`). The bulk import API expects LF. If bulk imports fail on row 1, use Apex Anonymous as alternative for small datasets.
- **Path separators**: Always use forward slashes (`/`) in `sf` CLI commands even on Windows — the SF CLI handles this correctly.
- **`force-app/` doesn't exist yet?** Run first-time setup: see `SETUP.md` in the repo root.
- **`sf` not found?** No admin install needed — see `SETUP.md` for the portable install option.
