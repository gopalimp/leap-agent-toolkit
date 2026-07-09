---
name: shared
description: Shared interactive input binding contract for all skills. Defines Mechanism 1 (ask_user_input_v0 for selections) and Mechanism 2 (structured plain-text prompts for free-text/numeric fields).
---

# Interactive Input Binding (Claude Desktop)

Claude Desktop exposes ONE interactive-input tool: **`ask_user_input_v0`**. It renders **tappable option buttons only** (single-select / multi-select). It does NOT support free-text fields, numeric fields, or grouped forms with text inputs.

So you have TWO mechanisms in Claude Desktop:

1. **`ask_user_input_v0`** — for SELECTIONS only (category, type, repository, entities, Yes/No).
2. **A structured plain-text prompt** — for any FREE-TEXT or NUMERIC value (host, port, database name, username, password). Send ONE message containing a fillable code block, then parse the user's reply.

You MUST pick the right mechanism for each input. NEVER stall after a tool call — if the next step requires text input, immediately send the structured plain-text prompt.

---

## Mechanism 1 — `ask_user_input_v0` (selections only)

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

## Mechanism 2 — Structured plain-text prompt (text / numeric / grouped fields)

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

## Choosing mechanism per step (create-repository example)

| Step | Input needed | Mechanism |
|---|---|---|
| 1a | Pick category | `ask_user_input_v0` (single-select) |
| 1b | Pick database type | `ask_user_input_v0` (single-select) |
| 2 | (none — tool call only) | — |
| 3 | Connection fields (text + number) | **Structured plain-text prompt** |
| 4 | Confirm retry on verify failure | `ask_user_input_v0` (Yes/No) |
| 5 | (none — tool call only) | — |

---

## Critical: never stall

If the next step needs text input and `ask_user_input_v0` cannot render it, you MUST immediately send the structured plain-text prompt described in Mechanism 2. Do NOT end your turn after a tool call without either (a) another tool call, (b) an `ask_user_input_v0` selection, or (c) a Mechanism-2 prompt.
