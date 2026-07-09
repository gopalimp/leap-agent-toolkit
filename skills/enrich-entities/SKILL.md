---
name: enrich-entities
description: Enrich entities (tables) with AI-generated descriptions and PII detection. Guides through datasource selection, entity selection (or bulk select), and enrichment job submission. The provider credential (Anthropic API key) is collected once per user through a secure browser form using MCP URL Mode Elicitation — never through chat.
---

# Enrich Entities

You are assisting the user in enriching entities with AI-generated descriptions and PII detection in their LeapLogic Metadata Catalog.

> **RULE #1: You MUST use `MCP elicitation/create` for ALL user input. NEVER ask questions in plain text.**
> **See the "Interactive Input Binding" section at the bottom of this file for concrete tool-call shapes (single-select, multi-select, grouped form, etc.).**
> **RULE #2: NEVER ask the user for their Anthropic API key or any other credential in chat.** Sensitive credentials are collected by the MCP server itself via URL Mode Elicitation. If the tool returns a URL prompt, surface that link to the user verbatim — do not attempt to capture, request, or echo the credential in any way.
> **RULE #3: The provider for this plugin is fixed to `anthropic`.** Always pass `provider="anthropic"` when calling `enrich_entities`. Never substitute a different provider.

## Significance

The **Enrich Entities** tool submits a background AI job that:
- Generates natural-language **descriptions** for each selected table and all its columns
- Detects and labels **PII (Personally Identifiable Information)** fields automatically
- Enriches metadata quality so data consumers can discover, understand, and govern data assets

This is essential for building a trusted, self-documenting data catalog.

### Before vs After Enrichment

- **Before enrichment**: When a user lists entities via `get_datasource_entities` or views entity details via `get_entity_details`, the `Description`, `Tags`, and column `Comments`/`Tags` fields will display **"None"** — indicating no metadata has been generated yet.
- **After enrichment**: These same fields will be populated with AI-generated descriptions, comments, and PII/classification tags, providing clear evidence that enrichment was successful.

## Provider context for this plugin

This plugin is wired to the **`anthropic`** AI provider. The model is chosen by the user from a curated list (see Step 4). The credential for this provider (your **Anthropic API key**) is required by the server, but it is **never** passed as a tool argument, request header, or chat message. Instead:

1. The first time `enrich_entities` is invoked for this provider, the server detects that no credential is stored for the user and triggers an **MCP URL Mode Elicitation** (spec 2025-11-25 §3.3).
2. The MCP client displays a secure URL plus a short explanation.
3. The user opens the URL in their browser, enters the credential into a server-hosted form, and submits.
4. The server stores the value encrypted at rest, keyed to the user's identity, and resumes the original `enrich_entities` call automatically.
5. On every subsequent call the stored credential is reused. The user is never prompted again unless they revoke it.

Your job is to **let this flow happen** — present any URL the tool returns to the user and instruct them to complete the form in a browser. Do NOT collect, request, or echo the credential in chat.

## Capabilities

### Enrich Selected Entities

Submit an enrichment job for specific entities chosen by the user from a multi-select list.

### Enrich All Entities in a Data Source

If the user says "enrich all tables", "enrich all entities", or "enrich the whole datasource / whole ds", automatically collect every entity ID and submit them all — no individual selection required.

## Workflow

### Step 1: Identify the Data Source

If the user hasn't already navigated to a specific data source:
1. Call `get_repositories` → present repos as a **single-select**
2. Call `get_datasources(repo_id=...)` → present data sources as a **single-select**

If enrichment was just performed in this session (e.g., immediately after creating a datasource), reuse the already-known `source_id` — do NOT repeat Steps 1–2.

### Step 2: Get Entity List

Call `get_datasource_entities(source_id=..., start=0, end=30)` to retrieve entities.

The tool returns up to **30 entities per call**. If the response contains **"Results may be paginated"**, more entities exist.

- **For "Enrich all" intent (Path A):** Automatically fetch all pages by calling `get_datasource_entities` in a loop — increment `start`/`end` by 31 each time (e.g., `start=0,end=30` → `start=31,end=61` → `start=62,end=92` → …) until no pagination note appears. Collect ALL entity IDs across all pages silently without prompting the user.
- **For "Enrich selected" intent (Path B):** Show the first page of entities to the user. If more exist, prompt them: "There are more entities. Would you like to see the next batch?" (using `MCP elicitation/create` with Yes/No options). Accumulate entities across pages, then present the full accumulated list as a multi-select.

### Step 3: Select Entities to Enrich

**Path A — "Enrich all" intent**

Trigger phrases (examples): "enrich all tables", "enrich all entities", "enrich the whole ds", "enrich everything", "enrich all".

- Do NOT show a selection UI
- Automatically collect ALL entity IDs from all paginated calls in Step 2
- Proceed directly to Step 4

**Path B — "Enrich specific tables" intent**

Trigger phrases (examples): "I want to enrich the tables", "let me pick which tables", "enrich some entities".

- Present entities as a **multi-select** dropdown using `MCP elicitation/create`:

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "Which tables/views do you want to enrich with AI descriptions and PII detection?",
    "requestedSchema": {
      "type": "object",
      "required": [
        "entities"
      ],
      "properties": {
        "entities": {
          "type": "array",
          "title": "Select Entities to Enrich",
          "items": {
            "type": "string",
            "enum": [
              "CUSTOMER_ORDERS (ID: 130)",
              "PRODUCT_CATALOG (ID: 131)",
              "USER_PROFILES (ID: 132)"
            ]
          }
        }
      }
    }
  }
}
```

- Build options dynamically from `get_datasource_entities` output — do NOT hardcode
- Extract the numeric ID from each selected label (e.g. `"CUSTOMER_ORDERS (ID: 130)"` → `130`)

### Step 4: Select Model

Present the curated model list for the `anthropic` provider as a **single-select** using `MCP elicitation/create`. Use the JSON block below **verbatim** — these are the only models supported by the LeapLogic backend for this plugin's provider:

```json
{
  "method": "elicitation/create",
  "params": {
    "message": "Which AI model should run the enrichment?",
    "requestedSchema": {
      "type": "object",
      "required": [
        "model"
      ],
      "properties": {
        "model": {
          "type": "string",
          "title": "Select Model",
          "oneOf": [
            {
              "const": "claude-sonnet-4-5-20250929",
              "title": "Claude Sonnet 4.5",
              "description": "Default \u2014 strong balance of quality and cost"
            },
            {
              "const": "claude-opus-4-1-20250805",
              "title": "Claude Opus 4.1",
              "description": "Highest quality reasoning"
            },
            {
              "const": "claude-haiku-4-5-20251022",
              "title": "Claude Haiku 4.5",
              "description": "Fastest, lowest cost"
            }
          ]
        }
      }
    }
  }
}
```

- The user's choice maps directly to a wire-level model identifier — pass it as the `model` argument in Step 5.
- If the user does not express a preference, default to the **first option** in the list (recommended default).

### Step 5: Submit Enrichment Job

Call `enrich_entities` with the entity IDs, the **fixed provider for this plugin**, and the **user-selected model id** — no credentials of any kind:

```
enrich_entities(entity_ids="130, 131, 132", provider="anthropic", model="<id from Step 4>")
```

If this is the first time the user has invoked the tool for this provider (or they previously revoked their credential), the MCP client will surface a URL Mode Elicitation prompt:

> **Open this link in your browser to provide your Anthropic API key:**
> `http://<mcp-server>/elicit/credential/<credential-key>?elicitationId=…`

In that case:
- Present the message and the URL **verbatim** to the user.
- Tell them: "Open the link in your browser, paste your Anthropic API key into the secure form, and click Save. I'll continue automatically once you've submitted it."
- Do NOT ask for the credential in chat.
- Do NOT retry the tool call until the user confirms they completed the form (the MCP client will resume the original call on its own once the form is submitted).

### Step 6: Display Result

Show the job submission result to the user exactly as returned by the tool:
- Job ID
- Status
- Number of entities submitted
- Message from the server

Inform the user that the enrichment job runs in the background and results will be available once processing completes.

**Post-enrichment verification:** Suggest the user call `get_datasource_entities` or `get_entity_details` after the job completes to verify that `Description`, `Tags`, and column `Comments` fields — which previously showed "None" — are now populated with AI-generated values.

## Examples

**User: "I want to enrich the tables"**
→ Show multi-select of all tables from the datasource → user picks → show model single-select → user picks model → call `enrich_entities` with selected IDs + `provider="anthropic"` + selected model id.

**User: "Enrich all tables" / "Enrich all entities" / "Enrich the whole ds"**
→ Skip entity selection → collect ALL entity IDs → show model single-select → user picks model → call `enrich_entities` with all IDs + `provider="anthropic"` + selected model id.

**Tool call for specific entities (IDs 130, 131) with the first model from the list:**
```
enrich_entities(entity_ids="130, 131", provider="anthropic", model="<first id from Step 4 list>")
```

**First-time invocation for this provider — server elicits the credential:**

The MCP client will display a message similar to:

> AI enrichment requires your Anthropic API key. Open the secure link below in a browser and enter the value. It is stored encrypted on the LeapLogic MCP server and never appears in chat history or the language model context.
>
> http://192.168.x.y:8778/elicit/credential/<credential-key>?elicitationId=…

Forward this verbatim to the user and wait for them to submit the form. Do NOT prompt them in chat.

## Error Handling

- If `entity_ids` contains non-numeric values: inform the user and ask for valid IDs.
- If no entities are selected (Path B): prompt user to select at least one entity.
- If authentication fails: advise refreshing the IDWSSOTOKEN.
- If the tool reports **"interactive prompt was declined, cancelled, or timed out"**: the user did not complete the browser form. Ask them to re-issue the original request and complete the secure link this time.
- If the tool reports **"Unsupported provider"** or **"Unsupported model"**: you almost certainly substituted a value not allowed by this plugin. Re-run with `provider="anthropic"` and a model from the Step 4 list.
- If the LeapLogic API returns an error: display the full error message to the user.

## Important Rules

- Always call `get_datasource_entities` to get current entity IDs — never assume or cache IDs from previous sessions.
- **Path A (enrich all)**: submit every entity ID from the datasource — no entity selection UI (model selection is still required).
- **Path B (enrich selected)**: always present a multi-select for entities — never single-select.
- **NEVER** ask the user for a Anthropic API key, an Anthropic API key, a GitHub Copilot token, or any other credential in chat. The server handles credential collection via URL Mode Elicitation.
- The `provider` argument is fixed to **`anthropic`** for this plugin — never override it.
- The `model` argument MUST be one of the `id` values from the Step 4 options list — never invent or substitute a model identifier.
- The enrichment job is asynchronous — inform the user that descriptions and PII labels will appear after processing.

## Interactive Input Binding (Claude Desktop)

Claude Desktop exposes ONE interactive-input tool: **`ask_user_input_v0`**. It renders **tappable option buttons only** (single-select / multi-select). It does NOT support free-text fields, numeric fields, or grouped forms with text inputs.

So you have TWO mechanisms in Claude Desktop:

1. **`ask_user_input_v0`** — for SELECTIONS only (category, type, repository, entities, Yes/No).
2. **A structured plain-text prompt** — for any FREE-TEXT or NUMERIC value (host, port, database name, username, password). Send ONE message containing a fillable code block, then parse the user's reply.

You MUST pick the right mechanism for each input. NEVER stall after a tool call — if the next step requires text input, immediately send the structured plain-text prompt.

---

### Mechanism 1 — `ask_user_input_v0` (selections only)

| Contract Primitive | `ask_user_input_v0` shape |
|---|---|
| Single-select | One question with an `options` array, `allowFreeformInput: false` |
| Multi-select | One question with `options` + `multiSelect: true` + `allowFreeformInput: false` |
| Boolean / Yes-No | One question with `options: [{"label":"Yes"},{"label":"No"}]`, `allowFreeformInput: false` |
| Confirmation | Same as Yes/No |

**Example — single-select after `get_repository_types`:**

```json
{
  "questions": [{
    "header": "repo_category",
    "question": "Select the repository category",
    "allowFreeformInput": false,
    "options": [
      {"label": "EDW", "description": "Category: RDBMS"},
      {"label": "Cloud Platform", "description": "Category: CLOUDPLATFORM"},
      {"label": "File System", "description": "Category: FILESYSTEM"}
    ]
  }]
}
```

Build option lists DYNAMICALLY from the prior tool's output — never hardcode.

---

### Mechanism 2 — Structured plain-text prompt (text / numeric / grouped fields)

After `get_source_fields` (or any step that needs free-text or numeric values), you MUST send a SINGLE chat message containing:

1. A short instruction line.
2. A fenced code block listing every required field as `key: <hint>` on its own line.
3. A request that the user paste the block back filled in.

You do this because `ask_user_input_v0` cannot render text fields. This is the supported workaround in Claude Desktop today.

**Template after `get_source_fields` returns the field list:**

> Please fill in the following connection details and send them back as one message:
>
> ```
> repo_name:     <Repository Name — alphanumeric, max 50 chars>
> host:          <Host Address>
> port:          <Port Number, default 1025>
> database_name: <Database Name — alphanumeric, max 64 chars>
> user_name:     <Username>
> password:      <Password>
> ```

**Hard rules for this mechanism:**

1. Use ONE message — never ask field-by-field across turns.
2. Include EVERY required field from `get_source_fields` (skip optional unless the user asks).
3. After the user replies, parse `key: value` lines case-insensitively, trim whitespace, then call the next tool (`verify_repository` → `create_repository`, etc.) with the parsed values.
4. If a value is missing or invalid, send the same fillable block again with the bad fields highlighted.
5. For SENSITIVE PLUGIN-INTERNAL CREDENTIALS in connection forms (e.g. database password): tell the user it will be sent to the MCP server and stored only inside the LeapLogic connection metadata. For PROVIDER credentials (Anthropic API key, GitHub Copilot token) the MCP server elicits via URL Mode — NEVER ask in chat.

---

### Choosing mechanism per step (create-repository example)

| Step | Input needed | Mechanism |
|---|---|---|
| 1a | Pick category | `ask_user_input_v0` (single-select) |
| 1b | Pick database type | `ask_user_input_v0` (single-select) |
| 2 | (none — tool call only) | — |
| 3 | Connection fields (text + number) | **Structured plain-text prompt** |
| 4 | Confirm retry on verify failure | `ask_user_input_v0` (Yes/No) |
| 5 | (none — tool call only) | — |

---

### Critical: never stall

If the next step needs text input and `ask_user_input_v0` cannot render it, you MUST immediately send the structured plain-text prompt described in Mechanism 2. Do NOT end your turn after a tool call without either (a) another tool call, (b) an `ask_user_input_v0` selection, or (c) a Mechanism-2 prompt.
