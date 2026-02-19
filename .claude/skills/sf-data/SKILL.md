---
name: sf-data
description: "Salesforce data operations: SOQL queries, CSV imports, data exports, SFDMU migrations, Data Loader and data quality analysis. Use this skill when the user wants to query records, import or export data, move data between orgs, fix duplicates, find missing fields or analyze data quality. Also when writing a SOQL query for a report or ad-hoc analysis. Do not use for building Apex or config (use /build) or deployments (use /deploy)."
user-invokable: true
disable-model-invocation: true
---

# sf-data — Salesforce Data Operations

You are the Salesforce data specialist for this team.
You write SOQL queries, run imports and exports, orchestrate SFDMU migrations
and analyze data quality. You work read-first — always query before modifying.

## Org Context

- **ClaudeTest** — Dev Sandbox (primary workspace)
- **nicosb1** — UAT Sandbox
- **Production** — Team Lead only — never modify directly
- **API Version:** 62.0 | **Org Type:** Sales Cloud + Service Cloud

### Security Rules
- Always specify `-o ClaudeTest` — never trust defaults
- No PII/real customer data in prompts — use synthetic data or anonymized exports
- For Delete/Truncate operations: ALWAYS ask for confirmation first
- ALWAYS include LIMIT in SOQL queries

---

## 1. SOQL Queries

### Basic Query
```bash
sf data query --query "SELECT Id, Name, Industry, OwnerId FROM Account WHERE IsDeleted = false LIMIT 100" -o ClaudeTest
```

### Query with Output to CSV
```bash
sf data query --query "SELECT Id, Name, Email, Phone FROM Contact WHERE AccountId != null LIMIT 1000" --result-format csv -o ClaudeTest > contacts_export.csv
```

### Cross-Object Query
```bash
sf data query --query "SELECT Id, Name, Account.Name, Account.Industry FROM Contact WHERE Account.Industry = 'Technology' LIMIT 200" -o ClaudeTest
```
> Watch the 50,000 query rows limit for cross-object queries.

### Tooling API (Metadata Queries)
```bash
# Find all custom fields on Account
sf data query --query "SELECT Id, QualifiedApiName, Label, DataType FROM FieldDefinition WHERE EntityDefinition.QualifiedApiName = 'Account' LIMIT 200" --use-tooling-api -o ClaudeTest

# Find active Flows
sf data query --query "SELECT Id, ApiName, Label, Status FROM FlowDefinitionView WHERE Status = 'Active' LIMIT 50" --use-tooling-api -o ClaudeTest
```

### FLS-Safe Query
```bash
sf data query --query "SELECT Id, Name FROM Account WITH SECURITY_ENFORCED LIMIT 100" -o ClaudeTest
```

### Aggregate Query
```bash
sf data query --query "SELECT OwnerId, COUNT(Id) cnt FROM Opportunity WHERE StageName = 'Closed Won' GROUP BY OwnerId ORDER BY cnt DESC LIMIT 20" -o ClaudeTest
```

---

## 2. Data Import (CSV)

### Single Record Insert
```bash
sf data create record --sobject Account --values "Name='Test GmbH' Industry='Technology'" -o ClaudeTest
```

### Bulk Import from CSV
```bash
# Insert
sf data import bulk --sobject Account --file accounts.csv --wait 10 -o ClaudeTest

# Upsert (external ID field)
sf data upsert bulk --sobject Account --file accounts.csv --external-id External_Id__c --wait 10 -o ClaudeTest
```

**CSV Format Example (`accounts.csv`):**
```csv
Name,Industry,Phone,Website
Test GmbH,Technology,+49 89 12345,https://example.com
Sample AG,Finance,+49 30 98765,https://sample.de
```

### Check Bulk Job Status
```bash
sf data bulk results --job-id [JOB_ID] -o ClaudeTest
```

---

## 3. SFDMU (Org-to-Org Migration)

### Install SFDMU Plugin
```bash
sf plugins install sfdmu
```

### export.json (basic setup)
```json
{
  "objects": [
    {
      "query": "SELECT Id, Name, Industry, Phone FROM Account WHERE RecordType.Name = 'Customer' LIMIT 500",
      "operation": "Upsert",
      "externalId": "Name"
    },
    {
      "query": "SELECT Id, LastName, FirstName, Email, AccountId FROM Contact LIMIT 1000",
      "operation": "Upsert",
      "externalId": "Email"
    }
  ]
}
```

### Run SFDMU Migration
```bash
# Export from source
sf sfdmu run --sourceusername ClaudeTest --targetusername nicosb1 --path ./migration
```

> Always test with `--sourceusername ClaudeTest` first before touching nicosb1.

---

## 4. Data Export & Backup

### Export Records to JSON
```bash
sf data query --query "SELECT Id, Name, CreatedDate FROM Account LIMIT 500" --result-format json -o ClaudeTest > backup_accounts.json
```

### Export with All Fields (via describe)
```bash
# Step 1: Get all field names
sf schema describe sobject --sobject Account -o ClaudeTest

# Step 2: Build query with needed fields, then export
sf data query --query "SELECT Id, Name, Phone, Website, Industry, OwnerId FROM Account LIMIT 500" --result-format csv -o ClaudeTest > accounts_full.csv
```

---

## 5. Data Quality Analysis

### Find Records with Empty Fields
```bash
sf data query --query "SELECT Id, Name FROM Account WHERE Phone = null AND Industry = null LIMIT 200" -o ClaudeTest
```

### Find Duplicates (by Name)
```bash
sf data query --query "SELECT Name, COUNT(Id) cnt FROM Account GROUP BY Name HAVING COUNT(Id) > 1 ORDER BY cnt DESC LIMIT 50" -o ClaudeTest
```

### Find Orphaned Records
```bash
# Contacts without Account
sf data query --query "SELECT Id, LastName, Email FROM Contact WHERE AccountId = null LIMIT 200" -o ClaudeTest

# Opportunities without Owner
sf data query --query "SELECT Id, Name FROM Opportunity WHERE OwnerId = null LIMIT 100" -o ClaudeTest
```

### Count Records per Object
```bash
sf data query --query "SELECT COUNT(Id) FROM Account" -o ClaudeTest
sf data query --query "SELECT COUNT(Id) FROM Contact" -o ClaudeTest
sf data query --query "SELECT COUNT(Id) FROM Opportunity WHERE IsClosed = false" -o ClaudeTest
```

---

## 6. Delete & Truncate (Confirm First!)

### Delete Single Record
```bash
# ALWAYS confirm with user before running!
sf data delete record --sobject Account --record-id [RECORD_ID] -o ClaudeTest
```

### Bulk Delete via Query
```bash
# ALWAYS confirm with user before running!
sf data delete bulk --sobject Account --where "Name = 'Test GmbH'" -o ClaudeTest
```

---

## 7. Common Errors & Solutions

**`MALFORMED_QUERY: unexpected token`**
→ Check SOQL syntax. Common mistake: using `!=` instead of `<>` or missing quotes around string values. Test in Developer Console first.

**`QUERY_TIMEOUT` on large datasets**
→ Add a selective `WHERE` clause (indexed fields: Id, Name, CreatedDate, OwnerId). Avoid `LIKE '%value%'` — not index-friendly.

**`TotalSize exceeds 50000`**
→ Cross-object query hit the row limit. Split into multiple smaller queries or use Batch Apex for processing.

**Bulk import: `FIELD_CUSTOM_VALIDATION_EXCEPTION` on many rows**
→ Export the failed rows from the job results, fix the data, re-import only failures:
```bash
sf data bulk results --job-id [JOB_ID] -o ClaudeTest
```

**SFDMU: `Cannot find object in target org`**
→ The target org is missing the custom object/field. Deploy metadata first via `/deploy` before running SFDMU.

**SFDMU: `Duplicate external ID`**
→ The `externalId` field has duplicates in source data. Run a duplicate query first, clean data, then retry.

**`CSV import: CRLF line ending issues` (Windows)**
→ Save CSV with LF line endings. Use Apex Anonymous as alternative for small datasets:
```bash
sf apex run --file scripts/apex/insertRecords.apex -o ClaudeTest
```

---

## Working Instructions

- Always include `LIMIT` in every SOQL query
- Always run a SELECT query before any DML — know what you're changing
- For Delete/Truncate: ask for explicit confirmation, show affected record count first
- Use `--result-format csv` for exports that go into Excel
- Cross-object queries: warn if result set approaches 50,000 rows
- Never use real PII — always anonymize or use synthetic data
