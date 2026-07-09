---
name: create-repository
description: Create a new repository in the LeapLogic Catalog. Guides through type selection, field discovery, connection verification, and repository creation.
---

# Create Repository

You are assisting the user in creating a new repository in the LeapLogic Catalog.

> **Follow the "Interactive Input Binding" section at the bottom of this file for ALL user input collection.** Never write plain-text bullet lists or "Please provide…" prompts.

## Workflow

Follow these steps in exact order. Do NOT skip steps.

### Step 1: Discover Available Repository Types

Call the `get_repository_types` tool to fetch the current list of categories and types.

CRITICAL: Do NOT write any text response listing the categories. Instead, immediately present a **single-select** with:
- Title: "Repository Category"
- Description: "Select the repository category"
- Options: One entry per category returned by the tool (use the category name as label)

After the user selects a category, present another **single-select** with:
- Title: "Repository Type"
- Description: "Select the database type"
- Options: One entry per type within the chosen category (use type name as label, add `[metadata-ui-enabled]` in description where applicable)

Do NOT print categories or types as bullet points or text. The user must see a navigable selection UI ONLY.

### Step 2: Get Required Fields

Once the user selects a type, call `get_source_fields` with:
- `db_type_category`: The category (e.g., "CLOUDPLATFORM")
- `db_type`: The specific type (e.g., "SNOWFLAKE")
- `scope`: "REPO"

Present ONLY the REQUIRED fields to the user. Show optional fields only if asked.

### Step 3: Collect Field Values

Collect values for the required fields from the user in ONE interaction, following the **Interactive Input Binding** section at the bottom of this file. Do NOT split the fields across multiple turns. Do NOT print a passive bullet list of fields and then stop — you MUST either render a grouped form (where supported) or send a structured fillable plain-text prompt as described in the binding.

Only include fields that are REQUIRED based on `get_source_fields` output. Adapt the fields to match the actual required fields for the selected type.

Use this parameter mapping:

| Source Field Key | Tool Parameter |
|-----------------|----------------|
| sourceName | repo_name |
| host | host |
| port | port |
| databaseName | database_name |
| userName | user_name |
| password | password |
| schemaName | schema_name |
| warehouseName | warehouse_name |
| jdbcParam | jdbc_param |

### Step 4: Verify Connection (if testConnectionFlag = true)

If `get_source_fields` returned `testConnectionFlag: true`, call `verify_repository` with the collected values BEFORE creating the repository.

If verification fails:
- Show the error to the user
- Ask them to correct the values
- Retry verification

### Step 5: Create Repository

Call `create_repository` with all collected parameters:
- `repo_name` (required)
- `db_type_category` (required)
- `db_type` (required)
- All connection fields from Step 3

For DDL-based repos: user must upload a DDL file first. The `artifact_id` comes from the file upload response.

### Error Handling

- If any tool returns a FAILED status, show the error verbatim to the user
- Do NOT guess or fabricate connection parameters
- Always confirm with the user before retrying

### Important Rules

- Return ALL tool output to the user EXACTLY as-is — do not summarize or reformat
- Do NOT hardcode any field requirements — always use `get_source_fields` to discover them dynamically
- Default `is_connection_for_repo_enable` to `true` unless the user explicitly says otherwise

## Interactive Input Binding (Claude Desktop)

> See [`skills/shared/SKILL.md`](../shared/SKILL.md) for the full interactive input binding contract (Mechanism 1: selections via `ask_user_input_v0`, Mechanism 2: structured plain-text prompts for text/numeric fields).
