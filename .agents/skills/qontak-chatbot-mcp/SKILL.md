---
name: qontak-chatbot-mcp
description: Use the Qontak Chatbot MCP server safely and effectively for path metadata, tree reads, branch editing, bot response updates, publish or discard workflows, and documented Hub lookup or upload tools. Use this skill whenever a task involves Qontak Chatbot path, conversation, or Hub operations through MCP tools.
metadata:
  author: GitHub Copilot
  version: "2026.6.20"
  source: local
---

# Qontak Chatbot MCP Skill

This skill helps an agent operate the Qontak Chatbot MCP server through the connected MCP tool surface.

Use only the MCP tools named in this skill when handling chatbot path, conversation tree, or Hub tasks. Do not rely on repository files, frontend source code, backend source code, or direct REST calls to decide what to do.

This file is the primary routing guide. The reference files expand details, but an agent should be able to choose the correct MCP tool family, prepare safe payloads, and recover from common failures by following this document alone.

Terminology:
- the chatbot API and MCP tool names use `path`
- the frontend UI wording uses `Conversation`
- when a user says `Conversation`, treat that as path metadata and path-level editing unless they clearly mean one specific node inside the conversation tree

The server is not a raw REST wrapper. It follows a FE-first contract:
- use frontend-aligned payload families first
- let the server translate normalized input into v1, v2, or v3 API calls
- prefer user-oriented workflow tools over stitching raw graph mutations when a workflow already exists
- use the documented Hub defaults, ordering, and quirks captured in this skill bundle instead of inventing new query shapes

## Setup Snapshot

Assume these conditions before using the tools:
- the Qontak Chatbot MCP server is already connected in the client
- chatbot tools use `CHATBOT_API_BASE_URL` and Hub tools use `HUB_API_BASE_URL`
- chatbot auth uses `CHATBOT_API_TOKEN`, with optional refresh through `CHATBOT_API_REFRESH_TOKEN`
- optional runtime settings include `CHATBOT_API_TIMEOUT_SECONDS`, `HUB_API_TIMEOUT_SECONDS`, `HUB_API_CLIENT_CONFIG_USE_STAGING`, `CONTENT_TYPE_CACHE_TTL_SECONDS`, and `LOG_LEVEL`

Use these lightweight checks when needed:
- `list_content_types(enabled=true)` for chatbot connectivity
- `get_hub_client_config()` for Hub connectivity
- `get_path_detail(path_id)` when the caller already knows a valid path id

Do not spend time on install steps, shell commands, or workspace setup when this skill is imported elsewhere. If tools are unavailable, ask the user to connect the MCP server and provide the required environment variables.

## Core Concepts

- `path` in tool names maps to `Conversation` in UI wording
- `channel_integration_id` is a chatbot-local integer owner id used by path writes
- `channels[]` is the selected Hub integration list attached to the path
- `channel_integration_id` must be a valid chatbot-local integer id, but it does not need a 1:1 mapping to every item in `channels[]`
- ids returned by `list_hub_integrations` are Hub ids and must not be reused as `channel_integration_id`
- tree reads default to and should use `v3`: it is the only version that serves the tree diagram, so `v1` and `v2` tree reads are retired and now 404
- `v3` is also the only tree version that surfaces interactive and `whatsapp_flow` detail, so it is correct for branch verification, branch-repair, and ordinary reads alike
- every tool returns a normalized envelope with `ok`, `operation`, `result`, optional `warnings`, and `meta`
- when a write succeeds with warnings, the mutation can still be valid even if tree verification had to fall back
- `nlp.confidence_level` is a 0..1 float; the FE shows it as a 1..100 percent, so convert before writing (FE 90% -> `0.9`)

## Content Types and Response Capabilities

The FE conversation builder offers these response/content types: text, button, list, ai_assist, branch, whatsapp_flow, ai_agent, and voice. The MCP now covers all of them except where the FE option has no backend counterpart. Use this catalog to know when the MCP can replicate an FE flow and when it cannot.

What the MCP CAN build:
- text: the `content_text` content type on any bot response (root or child).
- button: a top-level `interactive.button` payload on a bot response. This now works on CHILD create via `create_bot_response`, not just on update.
- list: a top-level `interactive.list` payload on a bot response. This now works on CHILD create via `create_bot_response`, not just on update.
- media (image, document, video): sent as a `text` node carrying a top-level `attachments[]` array. Media is not a separate content type; it rides on a text node. Attachments now ride along on CHILD create too.
- CRM deal: a top-level `crm_deal` block, carried on CHILD create now and editable later with `update_crm_deal`.
- tag: a top-level `tag` list, carried on CHILD create now.
- reuse: pass `bot_response: { "id": <int>, "is_reuse": true }` on `update_bot_response` to LINK an existing bot response instead of authoring new content. (Reuse is an update operation — the v1 create endpoint ignores `bot_response`, so it is NOT available on `create_bot_response`.)
- whatsapp_flow: dedicated `create_whatsapp_flow` and `update_whatsapp_flow` tools. `next_intent_type` accepts `TEXT`/`BUTTON`/`LIST`/`AI_ASSIST`/`BRANCH`; it does NOT support `VOICE` (the backend rejects it).
- fallback: dedicated `create_fallback` (default fallback reply, `intent_type` `FALLBACK` or `AI_ASSIST`) and `update_fallback` tools.
- path-level knowledge: `create_path_knowledge_sources` binds knowledge at the path level (distinct from node-level ai_assist `knowledge_sources`).
- organization entity: `create_organization_entity` mints the id used as `organization_entity_id`.
- NLP training: `train_utterances` (persist keyword utterances; texts must be unique) and `generate_trigger_texts` (GPT-generate candidate texts).
- ai_assist: a real content type on any bot response. Pass `content.content_type_code="ai_assist"` plus the top-level `ai_api_knowledge` and/or `knowledge_sources` (siblings of `content`/`components`) to `create_bot_response` for a child node, or nest those families inside `components` for `create_root_reply`. The knowledge families are forwarded to the backend ai_assist branch, so the node is created with its knowledge wired.
- ai_agent: `create_ai_agent_response` (POST /v2/bot_responses with `contentable_type="ai_agent"`) to create, and `update_ai_agent_response` (PATCH /v3/bot-responses/{id}) to edit. The agent must already exist; resolve its `contentable_id` with `list_ai_agents` first, and pass the agent's `content_type_version`.
- branch/condition: `create_branch_condition` and `update_branch_condition`. The legacy v1 `api.response_conditions[].criterias[]` shape (API-driven routing, POST/PATCH /v1/bot_responses/branches) still works. A top-level `conditions[]` array now adds SCHEDULE, CHANNEL, and CUSTOMER_FIELD branch types (auto-routing to the v2 branches endpoint). This is DISTINCT from `add_branch`, which adds a conversational user-input tree branch.
- standalone user input: `create_user_input` adds a user input with no child reply (and supports OCR/multitype via `response_type` + `ai_knowledge_source_ocr_*`). `add_branch` remains the way to create a user input plus its child reply in one call.
- exit-condition: `create_exit_condition` (a user input with `response_type="exit_condition"`). `next_intent_type` (TEXT/BUTTON/LIST/BRANCH/AI_AGENT/WHATSAPP_FLOW) auto-creates the downstream intent.
- voice: not a separate node type. Configure it on a bot response with `update_bot_response` (it auto-routes to v3) by setting `conversation_closure` plus an audio `attachments[]` entry. `bot_type="voice"` remains a path-level setting.
- intra-node ordering: the `sequence` field orders elements WITHIN a node — button actions, list items, and branch response-conditions/criterias. This is already supported on the relevant create/update payloads.

What the MCP CANNOT build (honest limitations):
- tree-level node reordering / re-parenting: there is no backend endpoint to move a node to a new parent or change sibling order at the tree level. Tree topology is fixed at creation time through `previous_bot_response_id` / `previous_user_input_id`. Only intra-node `sequence` (above) exists. If a caller needs to re-parent, that is a backend change, not an MCP gap.
- voice `action` (assign_to_call_group / end_the_call): the FE renders this dropdown but never sends it to the backend (it is hardcoded to a no-op on save). There is no backend counterpart, so the MCP intentionally omits it.
- delete-path: there is no `delete_path` tool (intentionally out of scope for this MCP). Path deletion must be done in the FE.

When a user wants one of the unsupported items above, say so plainly: the MCP cannot reproduce that part of the FE flow.

## Use This Skill When

Use this skill when the task involves any of these operations:
- read path metadata or conversation tree structure
- inspect one node and its direct children
- create or update a path or Conversation
- initialize a root reply
- add a branch under an existing bot response
- create or update a bot response node
- create or update a WhatsApp Flow bot response node
- update or delete user inputs
- publish or discard path drafts
- choose between direct mutation tools and higher-level workflow tools
- load Hub context such as profile, organization, integrations, divisions, tags, billing, client config, or Qontak uniq state
- upload Hub message files before attachment-oriented chatbot edits when the FE flow does that first

Do not use this skill for:
- direct frontend implementation work in Nuxt or Vue
- repository exploration or code search to infer payloads or defaults
- raw upstream REST payload design from scratch
- backend transport debugging that ignores the MCP normalization layer

## Operating Rules

1. Read first unless the task is a pure create flow.
2. Prefer `workflow_create_path_with_reply`, `workflow_add_branch_with_reply`, and `workflow_change_node_component` for common multi-step tasks.
3. Treat the current frontend behavior as the primary behavioral contract for conversation editing.
4. Use `create_root_reply` only for the seeded root node that already exists after path creation.
5. Use `create_bot_response` only for non-root child creation. It now carries top-level `interactive`, `attachments`, `crm_deal`, `tag`, `is_purchase_event`, and `organization_entity_id` on child create (a brand-new interactive is created inline through the v1 create path). To REUSE an existing bot response instead of authoring content, pass `bot_response: { "id": <int>, "is_reuse": true }` to `update_bot_response` — reuse is not accepted on create.
6. Use `update_bot_response` and `create_root_reply` to update interactive that already exists, matched by id, and for other advanced component edits such as knowledge sources, API actions, and conversation closure. Conversation closure and file-backed multipart attachments still require an update (not child create).
7. Inspect both `warnings` and `meta` on every response. They tell you which API and tree version were actually used.
8. Prefer safe delete defaults unless the user explicitly asks for broader deletion.
9. For Hub operations, prefer the documented default params and ordering in this skill bundle instead of inventing new query shapes.
10. Treat documented Hub quirks explicitly. Do not silently turn a client-config staging override or default-param mismatch into a global rule for every Hub tool.
11. Treat `channel_integration_id` as a chatbot-local integer id. Resolve it from `list_chatbot_channel_integrations` unless a trusted chatbot API read already supplied it.
12. Do not use ids from `list_hub_integrations` where chatbot write tools require `channel_integration_id`.
13. When the user asks to create or update a Conversation, treat that as `create_path`, `update_path_metadata`, or `workflow_create_path_with_reply` depending on whether they mean metadata only or metadata plus the first reply.
14. For Conversation create or update, keep the two channel concepts separate: `channel_integration_id` is the chatbot-local owner id, while `channels[]` is the selected Hub integration list attached to the Conversation.
15. `channel_integration_id` only needs to be a valid chatbot-local integer accepted by path CRUD. Do not force a 1:1 mapping between that owner id and every selected item in `channels[]`.
16. Keep Hub defaults conservative: `list_hub_users` defaults to `offset=1, limit=10`; `list_hub_integrations`, `list_hub_divisions`, and `list_hub_tags` default to `offset=1, limit=100`; `list_hub_tags` also defaults `query=""`.
17. Treat client-config staging as a tool-specific quirk, not a universal Hub base-url override.
18. Treat `workflow_create_path_with_reply` as non-atomic: a failed root-reply step can still leave a newly created path behind.
19. After a failed create workflow, verify whether a path was already created before retrying; otherwise retries can create duplicates.
20. When Hub integration lookup tools are unavailable, recover `channels[]` from a trusted existing path read or `list_paths` response by using response `channels[].integration_id` as request `channels[].id`.
21. Treat `add_branch` and `workflow_add_branch_with_reply` as non-idempotent: ambiguous failures can still create the `user_input` or the full branch, and blind retries can trigger duplicate-input errors.
22. When branch creation results are ambiguous, verify with `get_node_detail` on the parent bot response first and then use `get_path_tree(preferred_tree_version="v3")` as the primary graph check.
23. If branch creation returns `Created user input did not include an id`, treat it as a likely partial success, not a hard rollback.
24. For that partial-success error, verify in this order: `get_node_detail` on parent -> `get_path_tree(preferred_tree_version="v3")`. `v3` is the authoritative graph check; a `v2` tree re-read is a legacy no-op fallback now that the `v2` tree route 404s.
25. If `user_input` exists without child `bot_response`, repair by calling `create_bot_response` with parent `{ "type": "user_input", "id": ... }` and `preferred_tree_version="v3"`.
26. Before creating a branch, preflight with `get_node_detail` on the parent and check whether the target `user_input.input` already exists.
27. For multi-branch creation (for example numbered menus), run one branch at a time and complete verification or repair for each branch before moving to the next.
28. Never blind-retry `add_branch` or `workflow_add_branch_with_reply` after `Created user input did not include an id`; first detect whether the `user_input` already exists on the parent.
29. Treat a branch as complete only when both nodes exist: the `user_input` under the parent and its child `bot_response`.
30. Place advanced families by write level. For NON-ROOT writes (`create_bot_response`, `update_bot_response`, `workflow_change_node_component`), the fields `interactive`, `attachments`, `crm_deal`, `conversation_closure`, `ai_api_knowledge`, `knowledge_sources`, `tag`, and `content_type_version` are TOP-LEVEL siblings of `content`/`components`; `components` accepts ONLY `assignment`, `auto_resolve`, `idle_rule`, and `api`. For ROOT writes (`create_root_reply` and the `root_reply` of `workflow_create_path_with_reply`), those same advanced families nest INSIDE `components`. See references/component-updates.md for examples.

## Quick Start

### 1. Verify tool availability

- Use `list_content_types(enabled=true)` to confirm chatbot tool availability.
- Use `get_hub_client_config()` to confirm Hub tool availability when the task touches Hub state.

### 2. Pick the right tool family

- Read tools: `list_paths`, `get_path_detail`, `get_path_tree`, `get_node_detail`, `list_chatbot_channel_integrations`, `list_content_types`
- Workflow tools: `workflow_create_path_with_reply`, `workflow_add_branch_with_reply`, `workflow_change_node_component`
- Direct mutation tools: `create_path`, `update_path_metadata`, `create_root_reply`, `create_bot_response`, `add_branch`, `create_user_input`, `create_fallback`, `update_fallback`, `create_branch_condition`, `update_branch_condition`, `create_exit_condition`, `create_ai_agent_response`, `update_ai_agent_response`, `update_bot_response`, `update_crm_deal`, `create_whatsapp_flow`, `update_whatsapp_flow`, `update_user_input`, `set_default_user_input`, `create_path_knowledge_sources`, `create_organization_entity`, `train_utterances`, `generate_trigger_texts`, `delete_user_input`, `delete_bot_response`, `publish_path`, `discard_path_draft`
- AI agent lookup: `list_ai_agents` (resolve `contentable_id` before `create_ai_agent_response` / `update_ai_agent_response`)
- Hub tools: `get_hub_user_profile`, `list_hub_users`, `get_hub_organization_me`, `list_hub_integrations`, `normalize_hub_integrations_to_path_channels`, `list_hub_divisions`, `list_hub_tags`, `upload_hub_message_file`, `get_hub_qontak_integration_uniq`, `get_hub_billing_info`, `get_hub_client_config`

### 3. Prepare normalized payloads

- Resolve `channel_integration_id` with `list_chatbot_channel_integrations` before path create or path update.
- Convert selected Hub integrations into minimal `channels[]` items with `normalize_hub_integrations_to_path_channels`.
- Keep payloads on the normalized shapes documented in this skill bundle.
- If Hub integration tools are disabled, derive the write payload from a known good read: use response `channels[].integration_id` as request `channels[].id`, preserve `name`, and preserve `target_channel`.

Minimal `channels[]` item:

```json
{
   "id": "hub-integration-uuid",
   "name": "QontakSandbox5",
   "target_channel": "telegram"
}
```

Minimal `ParentRef`:

```json
{
   "type": "user_input",
   "id": 789
}
```

### 4. Follow a scenario workflow

- Read state first with `get_path_detail`, `get_path_tree`, and `get_node_detail` when the task is node-scoped.
- Prefer `workflow_create_path_with_reply` for path plus first reply.
- Prefer `workflow_add_branch_with_reply` for branch plus next reply.
- Prefer `workflow_change_node_component` for existing-node component edits.
- Use `publish_path` or `discard_path_draft` only when the task explicitly asks for that lifecycle step.

### 5. Handle component-specific edits correctly

- Use `update_bot_response` or `workflow_change_node_component` to update existing interactive buttons or lists (matched by id), attachments, CRM payloads, API actions, knowledge-source payloads, AI API knowledge, and conversation closure.
- Use `create_bot_response` for non-root child creation; it can also create a brand-new interactive inline through the v1 create path.

### 6. Check failure modes

- Inspect `ok`, `error`, `warnings`, `meta.api_version_used`, and `meta.tree_version_used` on every response.
- When a write fails, re-read with `get_path_detail`, `get_path_tree`, or `get_node_detail` before retrying.

## Payload And Transport Rules

Use these rules without exception unless the user explicitly wants a lower-level step.

### Path metadata

- use `create_path` for metadata-only creation
- use `update_path_metadata` for metadata-only updates
- always include `changes.channel_integration_id` on `update_path_metadata`
- `channel_integration_id` must be valid for the chatbot org, but it does not need to map 1:1 to the selected `channels[]` entries
- if the channel list came from Hub data, normalize it with `normalize_hub_integrations_to_path_channels` before writing
- for `channels[]` items in path writes, use Hub integration UUIDs as the `id` field (string format like "fdfb5512-45d2-4f4e-b2fd-710f4176a17a")
- do NOT include `integration_id` in path write payloads; response data includes it, but requests do not
- `operational_time_type` is required and must be one of: "24hours", "schedule", or "date_range"
- treat `operational_time_type` as a fixed path CRUD enum, not a lookup surface
- frontend path CRUD hardcodes `24hours`, `schedule`, and `date_range`, then submits the selected value directly on create and update
- backend path CRUD validates the same three values inline on `POST /v1/paths` and `PATCH /v1/paths/{id}`
- do not expect or invent a separate MCP tool, Hub tool, or upstream endpoint to list operational time types
- when `operational_time_type` is `schedule`, each schedule item must use `day_number` in `0..6`, `start_hour` and `end_hour` in `HH:MM:SS`, and `start_hour < end_hour`
- when `operational_time_type` is `date_range`, each item must use valid datetime strings and `start_date < end_date`
- keep `operational_time_type`, `schedules`, and `date_ranges` consistent after every update

### Root reply and child replies

- use `create_root_reply` only for the seeded root node that already exists after path creation
- use `create_bot_response` only for non-root child creation
- use `add_branch` when the task is explicitly low-level and needs both `user_input` and a child `bot_response`
- prefer `workflow_add_branch_with_reply` for the common branch-plus-reply flow
- for `workflow_create_path_with_reply`: always include a `name` field on `root_reply` (required; identifies the root bot response)
- for `workflow_create_path_with_reply`: prefer a simple safe root name like `Root` when upstream name validation is unclear; descriptive names may be rejected
- for `workflow_create_path_with_reply`: always include `is_send_message: true` in `root_reply.content` for user-facing messages
- for `add_branch`: always include `is_default` on `user_input` (required at branch creation time; exactly one branch per parent should have `is_default=true`)
- for `add_branch`: always include `channel_integration_id` on `user_input` (required; must be the chatbot-local integer id)

### Existing-node updates

- use `update_bot_response` or `workflow_change_node_component` for advanced edits on an existing bot response
- advanced edits include interactive buttons, interactive lists, attachments, CRM payloads, API actions, knowledge sources, AI API knowledge, assignment, idle rules, auto resolve, and conversation closure
- a brand-new interactive is created inline through `create_bot_response`; `update_bot_response` and `create_root_reply` update interactive that already exists, matched by id
- child create now also carries top-level `interactive`, `attachments`, `crm_deal`, `tag`, `is_purchase_event`, and `organization_entity_id`; only conversation closure and file-backed multipart attachments still require an update step. Reuse (`bot_response: { id, is_reuse: true }`) is carried by `update_bot_response`, not create
- use `update_crm_deal` to edit an existing CRM deal block (by `crm_deal_id`), and `update_fallback` to edit a fallback (always include `assignment`)
- if a payload is too advanced for child-create, create the child first and then update it

### Transport selection

- keep `transport="auto"` unless the task explicitly requires something narrower
- `transport="auto"` keeps ordinary JSON-safe edits on v2 JSON
- `transport="auto"` routes conversation closure to v3 JSON
- `transport="auto"` routes file-backed attachments to v3 multipart
- `transport="json"` should be used only when the caller intentionally wants JSON-only behavior

## Common Workflow Patterns

### Read a conversation safely

1. `get_path_detail(path_id)`
2. `get_path_tree(path_id, preferred_tree_version="v3")`
3. `get_node_detail(...)` only when the task targets a specific node or branch

### Create a conversation with the first message

1. `list_chatbot_channel_integrations`
2. `normalize_hub_integrations_to_path_channels` if Hub selections are involved
3. `workflow_create_path_with_reply`
4. If the workflow fails, inspect whether the path was still created before retrying
5. Use `list_paths` when you need to recover a newly created path by name or inspect existing channel metadata

### Update conversation metadata only

1. `get_path_detail`
2. `list_chatbot_channel_integrations` if `channel_integration_id` is missing or stale
3. `normalize_hub_integrations_to_path_channels` if Hub selections changed
4. `update_path_metadata`

### Add a branch with the next reply

1. `get_path_tree`
2. `get_node_detail` for the parent bot response
3. Pre-check whether the target `user_input.input` already exists on that parent
4. If missing, call `workflow_add_branch_with_reply` (or `add_branch` for low-level control)
5. If the result is ambiguous, re-read the parent bot response
6. If the parent read is still unclear, re-read the full tree with `preferred_tree_version="v3"` as the authoritative graph check (a `v2` tree re-read is a legacy no-op now that the `v2` tree route 404s)
7. If `user_input` exists but the child reply is missing, repair with `create_bot_response` on that `user_input` instead of recreating the branch
8. Mark the branch done only after both `user_input` and child `bot_response` are present

### Add multiple numbered branches smoothly

1. Read the parent once with `get_node_detail` and capture existing `user_input.input` values.
2. For each desired input (for example `1`, `2`, `3`, `4`), skip creation if it already exists.
3. Create one branch at a time with `workflow_add_branch_with_reply` or `add_branch`.
4. If `Created user input did not include an id` occurs, immediately re-read the parent in `v3` and locate the created input by exact `input` text.
5. If that input has no child reply, call `create_bot_response` against that `user_input`.
6. Continue to the next input only after the current input has a child `bot_response`.

### Change an existing node component

1. `get_node_detail`
2. `workflow_change_node_component` with `transport="auto"`
3. inspect `meta.api_version_used` and `warnings`

### Publish or discard draft changes

1. perform the required write tools first
2. `publish_path` only when the user asks to publish or the surrounding workflow clearly requires it
3. `discard_path_draft` only when the user explicitly asks to drop draft edits

## Hub Defaults And Ordering

Use these default payloads unless the caller specifies otherwise:

- `list_hub_users`: `{ "offset": 1, "limit": 10 }`
- `list_hub_integrations`: `{ "offset": 1, "limit": 100 }`
- `list_hub_divisions`: `{ "offset": 1, "limit": 100 }`
- `list_hub_tags`: `{ "offset": 1, "limit": 100, "query": "" }`

Use these lookup orders when the task does not specify a different sequence:

- auth bootstrap: `get_hub_client_config` -> `get_hub_user_profile` -> `get_hub_organization_me` -> `get_hub_billing_info`
- conversation setup: `list_hub_integrations` before path create or path update when channel selection matters
- assignment and tagging: `list_hub_tags` -> `list_hub_divisions` -> `list_hub_users`
- attachment flow: `upload_hub_message_file` before the chatbot node update that references the file
- CRM flow: `get_hub_qontak_integration_uniq` before CRM-related node edits

Important Hub quirks:

- `get_hub_client_config` may use a staging override controlled by MCP server configuration
- some callers attach tag-style params to `get_hub_organization_me`; do not require those params unless the user explicitly asks for parity with that caller behavior
- if a saved user or division id is missing from the first page, retry the same tool with `payload.query=<id>` and a narrow limit

## Decision Guide

Start with this decision tree:

1. Need to inspect state?
   Use `get_path_detail`, `get_path_tree`, `get_node_detail`, or `list_chatbot_channel_integrations`.
2. Need Hub profile, config, integrations, divisions, tags, uniq state, billing, or file upload context before an edit flow?
   Use the matching Hub tool and keep to the documented defaults in this skill bundle.
3. Need a valid `channel_integration_id` for path creation or path metadata updates?
   Use `list_chatbot_channel_integrations` and choose the chatbot-local integer `id`.
4. Need to turn selected Hub integrations into a minimal `channels[]` payload for a path write?
   Use `normalize_hub_integrations_to_path_channels`.
5. Need to create a path or Conversation and optionally set the first reply?
   Use `workflow_create_path_with_reply`.
6. Need to add a user choice plus the next reply under a bot response?
   Use `workflow_add_branch_with_reply`.
7. Need to edit an existing node's content or component payload?
   Use `workflow_change_node_component` or `update_bot_response`.
8. Need to create a simple non-root child reply under an existing node?
   Use `create_bot_response`.
9. Need to create or update a WhatsApp Flow bot response node?
   Use `create_whatsapp_flow` or `update_whatsapp_flow`.
10. Need to update only path or Conversation metadata?
   Use `update_path_metadata`.
11. Need to publish or discard draft changes?
   Use `publish_path` or `discard_path_draft`.
12. Need an ai_assist reply (knowledge-backed)?
   Use `create_bot_response` with `content.content_type_code="ai_assist"` plus top-level `ai_api_knowledge` / `knowledge_sources` (or `create_root_reply` with those nested in `components`).
13. Need an ai_agent reply node?
   Resolve the agent with `list_ai_agents`, then `create_ai_agent_response` with `contentable_id` and `content_type_version`. To edit an existing ai_agent node, use `update_ai_agent_response` (PATCH /v3/bot-responses/{id}).
14. Need a condition branch (route by API/criteria, not a user menu choice)?
   Use `create_branch_condition` / `update_branch_condition`. The legacy v1 `api.response_conditions` shape covers API routing; a top-level `conditions[]` adds SCHEDULE / CHANNEL / CUSTOMER_FIELD branches (auto-routes to v2). For a conversational user-input menu branch, use `add_branch` / `workflow_add_branch_with_reply` instead.
15. Need an exit-condition node?
   Use `create_exit_condition` with an `exit_condition` carrying `next_intent_type`. To UPDATE an existing exit condition's routing, use `update_user_input` with `response_type` + `parameters`.
16. Need a voice reply?
   Use `update_bot_response` on a bot response with `conversation_closure` + an audio `attachments[]` entry (auto-routes to v3). There is no separate voice node and no `action`/call-group field.
17. Need a standalone user input (no child reply) or an OCR/multitype input?
   Use `create_user_input` (`response_type` + `ai_knowledge_source_ocr_*` for OCR). Attach a reply later with `create_bot_response`.
18. Need a default fallback reply?
   Use `create_fallback` (`intent_type` `FALLBACK` or `AI_ASSIST`) to create, `update_fallback` to edit (always include `assignment`).
19. Need to edit a CRM deal on an existing node?
   Use `update_crm_deal` with the `crm_deal_id` and an `api_body` of deal fields. To attach CRM at creation, pass `crm_deal` on `create_bot_response`.
20. Need path-level knowledge, an organization entity id, or NLP keyword training?
   Use `create_path_knowledge_sources` (path-level knowledge), `create_organization_entity` (mint an `organization_entity_id`), `train_utterances` (persist unique utterances), or `generate_trigger_texts` (GPT-generate candidates).

## Tool Strategy

### Read tools

Use these to ground every mutation in actual tree state:
- `list_chatbot_channel_integrations`
- `get_path_detail`
- `get_path_tree`
- `get_node_detail`
- `list_content_types`

### Workflow tools

Prefer these when they fit the requested outcome:
- `workflow_create_path_with_reply`
- `workflow_add_branch_with_reply`
- `workflow_change_node_component`

### Hub tools

Use these when the FE flow reads Hub state before or alongside chatbot editing:
- `get_hub_user_profile`
- `list_hub_users`
- `get_hub_organization_me`
- `list_hub_integrations`
- `normalize_hub_integrations_to_path_channels`
- `list_hub_divisions`
- `list_hub_tags`
- `upload_hub_message_file`
- `get_hub_qontak_integration_uniq`
- `get_hub_billing_info`
- `get_hub_client_config`

### Direct mutation tools

Use these when the workflow wrapper is too broad or the task is explicitly low-level:
- `create_path`
- `update_path_metadata`
- `create_root_reply`
- `create_bot_response`
- `add_branch`
- `create_user_input`
- `create_fallback`
- `update_fallback`
- `create_branch_condition`
- `update_branch_condition`
- `create_exit_condition`
- `list_ai_agents`
- `create_ai_agent_response`
- `update_ai_agent_response`
- `update_bot_response`
- `update_crm_deal`
- `create_whatsapp_flow`
- `update_whatsapp_flow`
- `update_user_input`
- `set_default_user_input`
- `create_path_knowledge_sources`
- `create_organization_entity`
- `train_utterances`
- `generate_trigger_texts`
- `delete_user_input`
- `delete_bot_response`
- `publish_path`
- `discard_path_draft`

## Safe Recovery Pattern

When a write fails or routes unexpectedly:

1. inspect `error.type`, `error.message`, `warnings`, `meta.api_version_used`, and `meta.tree_version_used`
2. re-read with `get_path_detail`
3. re-read with `get_path_tree`
4. re-read the target node with `get_node_detail` if the task is node-scoped
5. verify whether the payload violated one of the core rules in this file
6. retry with the narrowest valid tool

Common causes:

- unresolved `content_type_code`: call `list_content_types(enabled=true)` and retry with a valid code
- missing `changes.channel_integration_id`: resolve it from `list_chatbot_channel_integrations`
- wrong parent reference: re-read the tree and retry with the live `ParentRef`
- Hub integration id used as `channel_integration_id`: replace it with the chatbot-local integer id
- multipart-only payload forced through JSON: switch back to `transport="auto"`
- root bot response targeted for deletion: update it instead of deleting it
- branch create returns `Created user input did not include an id`: treat as partial success, locate `user_input` by exact `input` text on the parent, then repair missing child reply with `create_bot_response` instead of retrying branch creation

## Response Handling

Expect a normalized response envelope:
- `ok`
- `operation`
- `path`
- `result`
- `warnings`
- `meta`
- `error` when `ok` is `false`

Always report:
- what tool was used
- which path or node changed
- whether warnings were returned
- which `api_version_used` and `tree_version_used` were reported when relevant

## Reference Use

Use the reference files for deeper detail, not as a substitute for the routing rules above.

- `references/setup.md`: MCP server configuration and environment variables
- `references/tool-catalog.md`: per-tool behavior and selection notes
- `references/hub-tools.md`: Hub defaults, lookup order, and quirks
- `references/payloads.md`: normalized payload examples
- `references/workflows.md`: expanded scenario walkthroughs
- `references/component-updates.md`: advanced bot-response component rules
- `references/troubleshooting.md`: deeper failure analysis

## Reference Index

- [references/setup.md](references/setup.md)
- [references/tool-catalog.md](references/tool-catalog.md)
- [references/hub-tools.md](references/hub-tools.md)
- [references/payloads.md](references/payloads.md)
- [references/workflows.md](references/workflows.md)
- [references/component-updates.md](references/component-updates.md)
- [references/troubleshooting.md](references/troubleshooting.md)
