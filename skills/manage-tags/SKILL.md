---
name: manage-tags
description: Add tags to entities (tables/views) or attributes (columns) in the metadata catalog. Helps organize and classify data assets.
---

# Manage Tags

You are assisting the user in tagging entities and attributes in their LeapLogic Metadata Catalog.

> **Follow the "Interactive Input Binding" section at the bottom of this file for ALL user input collection.** Never write plain-text bullet lists or "Please provide…" prompts.

## Capabilities

### Add Tags to an Entity (Table/View)

Tag a table or view with descriptive labels for classification and discovery.

### Add Tags to an Attribute (Column)

Tag a specific column with labels such as "pii", "sensitive", "finance", etc.

## Workflow

### Step 1: Identify the Target

If the user hasn't already navigated to a specific entity:
1. Call `get_repositories` → present repos as a **single-select**
2. Call `get_datasources` → present data sources as a **single-select**
3. Call `get_datasource_entities` → present entities as a **single-select**
4. Call `get_entity_details` → show columns and existing tags

For all selection steps above, use single-select with no freeform input allowed.

### Step 2: Select What to Tag

Present a **single-select** asking:
- Title: "Tag Target"
- Description: "What do you want to tag?"
- Options: "The entity (table/view) itself", "A specific column"

If column: present another **single-select** with column names from the entity details.

### Step 3: Collect Tags

Collect tags using a **text input** asking the user for tag names.
- Title: "Tags"
- Description: "Enter tags (comma-separated, each at least 3 characters)"
- Validate: each tag >= 3 characters before calling the API

### Step 3: Apply Tags

Call `add_tags` with:
- `resource_id`: The entity ID or attribute ID
- `resource_type`: "ENTITY" for tables/views, "ATTRIBUTE" for columns
- `resource_name`: The name of the entity or attribute
- `tags`: Comma-separated tag names

### Step 4: Verify

Call `get_entity_details` to confirm the tags were applied successfully.

## Examples

**Tagging a table:**
```
add_tags(resource_id=42, resource_type="ENTITY", resource_name="CUSTOMER_ORDERS", tags="finance, orders, production")
```

**Tagging a column:**
```
add_tags(resource_id=128, resource_type="ATTRIBUTE", resource_name="email_address", tags="pii, sensitive")
```

## Error Handling

- If a tag is less than 3 characters: inform the user and ask for a longer name
- If the resource ID is invalid: suggest using `get_entity_details` to find the correct ID
- If authentication fails: advise refreshing the IDWSSOTOKEN

## Important Rules

- Always show existing tags (from `get_entity_details`) before adding new ones
- Confirm with the user before applying tags
- Tags cannot be removed via this tool — only added

## Interactive Input Binding (Claude Desktop)

> See [`skills/shared/SKILL.md`](../shared/SKILL.md) for the full interactive input binding contract (Mechanism 1: selections via `ask_user_input_v0`, Mechanism 2: structured plain-text prompts for text/numeric fields).
