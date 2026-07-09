---
name: verify-connection
description: Test connectivity for an existing or new repository or data source connection. Diagnoses connection failures and suggests fixes.
---

# Verify Connection

You are assisting the user in testing database connections — either for existing resources or new configurations.

> **Follow the "Interactive Input Binding" section at the bottom of this file for ALL user input collection.** Never write plain-text bullet lists or "Please provide…" prompts.

## Use Cases

### 1. Verify an Existing Repository

1. Call `get_repositories` to list repos
2. Present repos as a **single-select** (no freeform input allowed)
3. Call `verify_repository` with the selected repo's connection details

### 2. Verify a New Repository Configuration

Collect connection parameters using a **grouped form**:
- First: single-select for `db_type_category` and `db_type`
- Then: grouped form with fields for connection (host, port), database (database_name, schema_name), auth (user_name, password)

Call `verify_repository` with collected values.

### 3. Verify an Existing Data Source

1. Call `get_repositories` → present as **single-select**
2. Call `get_datasources` → present as **single-select**
3. Call `verify_datasource` with the selected DS connection details

### 4. Verify a New Data Source Configuration

Collect DS parameters using a **grouped form**:
- `repo_id` (from repository single-select), `ds_name`, `database_name`, `schema_name`
- Plus type-specific fields

Call `verify_datasource` with collected values.

## Result Interpretation

### Success
```
Status: SUCCESS
Connection verified successfully.
```
Inform the user that connectivity is confirmed.

### Failure — Common Causes

| Error Pattern | Likely Cause | Suggestion |
|---------------|--------------|------------|
| Connection refused | Host/port wrong or firewall | Verify host:port, check firewall rules |
| Authentication failed | Bad username/password | Re-enter credentials |
| Database not found | Wrong database name | Check database name spelling |
| Schema not found | Wrong schema name | List available schemas |
| Timeout | Network issue | Check VPN, network connectivity |
| SSL/TLS error | Certificate issue | Check SSL config or disable SSL verification |

### Retry Logic

- After a failure, show the diagnosis and ask user for corrected values
- Re-run verification with the corrected values
- Do NOT create a repository/DS until verification passes

## Important Rules

- This skill is DIAGNOSTIC ONLY — it does not create or modify any resources
- Always show the full error message from the API — do not summarize
- If the user wants to proceed to creation after a successful verify, hand off to `create-repository` or `create-datasource` skill

## Interactive Input Binding (Claude Desktop)

> See [`skills/shared/SKILL.md`](../shared/SKILL.md) for the full interactive input binding contract (Mechanism 1: selections via `ask_user_input_v0`, Mechanism 2: structured plain-text prompts for text/numeric fields).
