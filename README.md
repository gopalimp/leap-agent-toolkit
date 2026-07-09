# LeapLogic Metadata Catalog Plugin

A Claude Code and MCP-compatible plugin for managing LeapLogic Metadata Catalog workflows, including repository onboarding, data source creation, metadata browsing, AI enrichment, PII detection, and tag-based governance.

## Features

- **Browse Catalog** — List repositories, data sources, and entities; explore entity details (columns, types, tags, indexes)
- **Create Repository** — Guided workflow: type selection → field discovery → verify → create
- **Create Data Source** — Guided workflow: repo selection → fields → verify → create → entities
- **Verify Connection** — Test connectivity for repos or data sources before creating them
- **Enrich Entities** — Submit an AI job to generate descriptions and detect PII for selected tables (or all tables in a data source)
- **Manage Tags** — Add tags to entities (tables/views) or attributes (columns) to organize and classify data assets

````md
### Currently Supported Repository Types

The plugin currently supports two repository categories:

- **Cloud** — Snowflake
- **EDW** — Teradata
````

## Prerequisites

- LeapLogic MCP server running with Keycloak OAuth configured
- A Dev or Full seat on a LeapLogic plan

## How Authentication Works

The MCP server implements the **MCP OAuth specification**. When you connect:

1. Your MCP client connects to the server
2. Server returns `401 Unauthorized` (no valid token yet)
3. Client discovers OAuth metadata at `/.well-known/oauth-authorization-server`
4. Client opens the **Keycloak login page** in your browser automatically
5. You log in with your LeapLogic credentials
6. Client receives an access token and sends it with all subsequent requests
7. Server auto-generates IDWSSOTOKEN internally — **no manual token management required**

## Installation & Setup

### Claude Code

The recommended way to set up the LeapLogic MCP server in Claude Code is by installing the plugin:

```bash
claude plugin install leaplogic-catalog@claude-community
```

**Manual setup:**

```bash
claude mcp add leaplogic-metadata --transport http https://test.leaplogic.io/mcp
```

### Claude Desktop

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "leaplogic-metadata": {
      "url": "https://test.leaplogic.io/mcp"
    }
  }
}
```

The server will trigger the Keycloak login flow automatically.

### Cursor

Install the plugin by typing the following command in Cursor's agent chat:

```
/add-plugin leaplogic-catalog
```

### VS Code

1. Use the shortcut `Ctrl+Shift+P` to search for `MCP: Add Server`.
2. Select `HTTP`.
3. Paste the server URL `https://test.leaplogic.io/mcp` in the search bar. Then hit `Enter`.
4. When prompted for a server ID, enter `leaplogic-metadata`.
5. Select whether to add globally or for the current workspace only.

### Other Editors

Any editor that supports Streamable HTTP can connect:

```json
{
  "mcpServers": {
    "leaplogic-metadata": {
      "url": "https://test.leaplogic.io/mcp"
    }
  }
}
```

## Configuration

| Setting | Required | Description |
|---------|----------|-------------|
| `mcp_server_url` | Yes | MCP server endpoint (e.g., `https://test.leaplogic.io/mcp`) |

That's it — no tokens to manage. Authentication is handled automatically via OAuth.

## Directory Structure

```
agent-toolkit/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── .cursor-plugin/
│   └── plugin.json          # Cursor plugin manifest
├── .mcp.json                # MCP server configuration (HTTP type)
├── server.json              # Optional MCP server metadata using the official server schema
├── settings.json            # Optional project metadata for internal/default agent selection
├── LICENSE                  # Apache-2.0
├── README.md                # This file
├── assets/
│   └── leaplogic-icon.svg   # Plugin icon
├── agents/
│   └── metadata-catalog.md  # Specialist agent definition
└── skills/
    ├── shared/
    │   └── SKILL.md  # Shared input binding contract
    ├── browse-catalog/
    ├── create-repository/
    ├── create-datasource/
    ├── enrich-entities/
    ├── manage-tags/
    └── verify-connection/
```

## Available Tools (via MCP)

| Tool | Description |
|------|-------------|
| `get_repository_types` | List supported repository types by category |
| `get_source_fields` | Discover required fields for a repo/DS type |
| `get_repositories` | List all registered repositories |
| `get_datasources` | List data sources in a repository |
| `verify_repository` | Test repository connectivity |
| `create_repository` | Register a new repository |
| `verify_datasource` | Test data source connectivity |
| `create_datasource` | Add a data source to a repository |
| `get_datasource_entities` | List entities (tables/views) in a data source |
| `get_sync_status` | Check import/sync progress of data sources |
| `get_entity_details` | Get column-level details, tags, and indexes for an entity |
| `add_tags` | Add tags to an entity (table/view) or attribute (column) |
| `enrich_entities` | Submit an AI job that generates descriptions and detects PII for selected entities |

## Best Practices

### Structure your catalog for effective AI exploration

- **Name entities semantically** — Use meaningful names for repositories, data sources, and schemas (e.g., `sales-warehouse` not `repo_01`)
- **Tag everything** — Tags improve discoverability; apply domain, sensitivity, and ownership tags early
- **Enrich early** — Run `enrich_entities` right after data source creation to populate descriptions and PII labels while context is fresh

### Write effective prompts

- Be specific about scope: "Show me all PII columns in the customer_orders table"
- Use catalog hierarchy: "List data sources in the production-warehouse repository"
- Chain operations: "Create a Snowflake repository, add a data source, then enrich all discovered tables"

### Workflow tips

1. **Always verify before creating** — The plugin enforces verify → create ordering. Connection issues are cheaper to fix before resource creation.
2. **Browse first** — Explore the catalog before making changes to understand existing structure.
3. **Bulk enrich** — Use "enrich all entities" for newly imported data sources rather than enriching one at a time.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Login page doesn't appear | Ensure Keycloak is reachable at configured host:port |
| "Connection refused" | Verify MCP server host/port and network access |
| Tools not loading | Confirm the MCP server is reachable and exposing the metadata catalog tools |
| Token expired mid-session | MCP client will auto-refresh; if not, reconnect |
| Skills not appearing | Run `/reload-plugins` in Claude Code or restart the client |

## License

[Apache-2.0](./LICENSE)
