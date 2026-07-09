---
name: create-datasource
description: Create a new data source within an existing repository. Guides through repository selection, field discovery, connection verification, and data source creation.
---

# Create Data Source

You are assisting the user in creating a new data source inside an existing LeapLogic repository.

> **Follow the "Interactive Input Binding" section at the bottom of this file for ALL user input collection.** Never write plain-text bullet lists or "Please provide…" prompts.

## Workflow

Follow these steps in exact order. Do NOT skip steps.

### Step 1: Select Repository

Call `get_repositories` to get the list of available repositories.

CRITICAL: Do NOT write text listing repositories. Immediately present a **single-select** with:
- Title: "Select Repository"
- Description: "Choose the parent repository for the new data source"
- Options: One entry per repository (label = repo name, description = "Type: {type} | ID: {sourceId}")

Build options dynamically from `get_repositories` output. Do NOT hardcode.

### Step 2: Discover Required Fields for Data Source

Call `get_source_fields` with:
- `db_type_category`: From the selected repo's category
- `db_type`: From the selected repo's type
- `scope`: "DS"

This returns the fields required specifically for adding a data source under this repository type.

### Step 3: Collect Field Values

Collect values for the required fields from the user in ONE interaction, following the **Interactive Input Binding** section at the bottom of this file. Do NOT split the fields across multiple turns. Do NOT print a passive bullet list of fields and then stop — you MUST either render a grouped form (where supported) or send a structured fillable plain-text prompt as described in the binding.

Only include fields that are REQUIRED based on `get_source_fields` output. Add optional fields marked as "(optional)" in their description.

Key parameters include:
- `repo_id`: The sourceId of the selected repository (from Step 1)
- `ds_name`: User-chosen name for the data source
- `database_name`: Target database
- `schema_name`: Target schema

Additional fields depend on the repository type (e.g., `warehouse_name` for Snowflake).

### Step 4: Verify Data Source Connection

Call `verify_datasource` with the collected values to test connectivity.

If verification fails:
- Display the error message to the user
- Ask them to correct the problematic values
- Retry verification until it succeeds or the user abandons

### Step 5: Create Data Source

Call `create_datasource` with all collected parameters.

On success, the tool returns the newly created `sourceId` and automatically triggers an import.
The import runs asynchronously — entities will be discovered in the background.

### Step 6: Check Import Status

Call `get_sync_status` with the new data source's `ds_id` to check if import has completed.

- If sync is still in progress: inform the user and suggest waiting before checking again.
- If sync is complete: proceed to Step 7.

### Step 7: View Discovered Entities

Once sync is complete, call `get_datasource_entities` with the new data source's `sourceId`.

Present the entity list with name, type (TABLE/VIEW), and row count.

Optionally, call `get_entity_details` for any entity the user wants to inspect further.

## Error Handling

- If `verify_datasource` fails with connection errors, suggest checking:
  - Network connectivity to the target database
  - Credentials (username/password)
  - Database/schema names are spelled correctly
  - Firewall rules allow access from the LeapLogic server
- If `create_datasource` fails, relay the full error message to the user

## Important Rules

- ALWAYS verify before creating — do NOT skip Step 4
- The `repo_id` must come from `get_repositories` — never let the user type an arbitrary ID
- Do NOT hardcode field requirements — always use `get_source_fields` with `scope=DS`

## Interactive Input Binding (Claude Desktop)

> See [`skills/shared/SKILL.md`](../shared/SKILL.md) for the full interactive input binding contract (Mechanism 1: selections via `ask_user_input_v0`, Mechanism 2: structured plain-text prompts for text/numeric fields).
