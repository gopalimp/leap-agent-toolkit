---
name: browse-catalog
description: Browse the metadata catalog — list repositories, data sources, and entities. Read-only exploration with no side effects.
---

# Browse Catalog

You are assisting the user in exploring their LeapLogic Metadata Catalog. This skill is read-only and makes no changes.

> **Follow the "Interactive Input Binding" section at the bottom of this file for ALL user input collection.** Never write plain-text bullet lists or "Please provide…" prompts.

## Capabilities

### List All Repositories

Call `get_repositories` to display all registered repositories.

Present results as a formatted table:
| # | Name | Type | Host | Database | Source ID |
|---|------|------|------|----------|-----------|

### List Data Sources in a Repository

When the user picks a repository (or provides a `sourceId`), call `get_datasources` with that `source_id`.

Present results:
| # | Name | Type | Database | Schema | Source ID |
|---|------|------|----------|--------|-----------|

### List Entities in a Data Source

When the user picks a data source, call `get_datasource_entities` with that `source_id`.

The tool returns up to **30 entities per call** (default pagination: `start=0, end=30`).

Present results with metadata about the data source (name, type, database, schema) followed by:
| # | Entity Name | Type | Entity ID | Description | Tags |
|---|-------------|------|-----------|-------------|------|

> **Note:** `Description` and `Tags` show "None" for entities that have not been enriched yet. After running the `enrich-entities` skill, these fields will display AI-generated descriptions and detected tags.

#### Pagination Flow

If the tool output contains **"Results may be paginated"**, there are more entities. Use `MCP elicitation/create` to prompt the user:

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "There are more entities in this data source. Would you like to see the next batch?",
    "requestedSchema": {
      "type": "object",
      "required": [
        "choice"
      ],
      "properties": {
        "choice": {
          "type": "string",
          "title": "More Entities Available",
          "oneOf": [
            {
              "const": "yes",
              "title": "Yes, show more"
            },
            {
              "const": "no",
              "title": "No, that's enough"
            }
          ]
        }
      }
    }
  }
}
```

- If **"Yes, show more"**: call `get_datasource_entities(source_id, start=31, end=61)` and display the next page. Repeat pagination check.
- If **"No, that's enough"**: stop and await further user instructions.
- Continue incrementing `start`/`end` by 31 for each subsequent page until user declines or no pagination note appears.
### View Entity Details

When the user picks an entity, call `get_entity_details` with the `entity_id`.

Present the entity metadata (name, type, database, schema, description, tags, domain) followed by columns:
| # | Column Name | Data Type | Nullable | PK | Comments | Tags |
|---|-------------|-----------|----------|----|----------|------|

> **Note:** Entity-level `Description` and `Tags`, as well as column-level `Comments` and `Tags`, always display "None" when not yet enriched. After enrichment, these fields will show AI-generated values.

Also show indexes if available.

### Check Sync Status

If the user asks about import progress or sync status, call `get_sync_status`.

- With `ds_id`: checks if that specific data source is still importing
- Without `ds_id`: shows an overview of all active syncs

If sync is complete, suggest calling `get_datasource_entities` to view results.

## Navigation Flow

```
Repositories → Data Sources → Entities → Entity Details
     ↑              ↑             ↑            |
     |______________|_____________|____________|
              (user can go back)
```

- Allow the user to navigate up/down the hierarchy freely
- Remember the current context (which repo/DS is selected) across turns
- When the user needs to select a resource from a list, present a **single-select** with:
  - Each item as an option with `label` = resource name, `description` = "ID: {sourceId} | Type: {type}"
  - No freeform input allowed — user must pick from the list
- Accept either a name or sourceId to identify a resource

## Formatting Rules

- **Always reformat tool output into Markdown tables** — tools return space-padded text which is unreadable in chat. Convert all tabular data to proper Markdown tables with `|` column borders and `|---|` header separators
- Show counts: "Found 5 repositories" / "Found 12 entities in data source X"
- Entity lists are paginated (30 per page) — follow the pagination flow above
- Highlight key info: connection status, database type, entity types
- Present metadata headers (entity name, type, database, schema, description, tags) as **bold key-value pairs** above the table
- Never output raw space-padded plain text to the user

## Error Handling

- If no repositories exist: inform the user and suggest using the `create-repository` skill
- If a repository has no data sources: inform and suggest `create-datasource`
- If authentication fails: advise refreshing the IDWSSOTOKEN

## Interactive Input Binding (Claude Desktop)

> See [`skills/shared/SKILL.md`](../shared/SKILL.md) for the full interactive input binding contract (Mechanism 1: selections via `ask_user_input_v0`, Mechanism 2: structured plain-text prompts for text/numeric fields).
