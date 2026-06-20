# Workflows

## Read a Path Safely

UI wording note:
- when the user says read or inspect a Conversation, use this same path-read sequence

Use this sequence before targeted edits.

1. Call `get_path_detail(path_id)`.
2. Call `get_path_tree(path_id, preferred_tree_version="v3")`.
3. If the task targets one branch or one node, call `get_node_detail(path_id, node_type, node_id)`.
4. Use returned `warnings` and `meta.tree_version_used` to understand fallback behavior before writing.

## Create a Path With the First Reply

UI wording note:
- in the frontend, this is create a Conversation with its first message

Preferred tool:
- `workflow_create_path_with_reply`

Recommended request:

```json
{
  "path": {
    "name": "Customer Service Main Path",
    "bot_type": "chat",
    "channel_integration_id": 123,
    "operational_time_type": "24hours",
    "channels": [
      {
        "id": "fdfb5512-45d2-4f4e-b2fd-710f4176a17a",
        "name": "QontakSandbox5",
        "target_channel": "telegram"
      }
    ]
  },
  "root_reply": {
    "name": "Root",
    "content": {
      "content_text": "Hello and welcome!",
      "content_type_code": "text",
      "channel_integration_id": 123,
      "is_send_message": true
    }
  }
}
```

Why this is preferred:
- it creates the path first
- it re-reads the tree
- it locates the seeded root bot response
- it initializes that root through the existing root flow

Before this workflow:
1. Resolve the chatbot-local `channel_integration_id` from `list_chatbot_channel_integrations`.
2. Resolve the Hub integration UUID for selected channels.
3. Include `operational_time_type` (required; use "24hours", "schedule", or "date_range").
4. Include channels array populated with resolved channel selections or empty if no channels.
5. Ensure `root_reply.name` is included (required; identifies the root bot response node).
6. Ensure `root_reply.content.is_send_message` is true (required for user-facing messages).
7. If Hub lookup tools are unavailable, recover channel UUIDs from a trusted path read and map response `channels[].integration_id` into request `path.channels[].id`.
8. Send `root_reply` in the full bot-response shape with a nested `content` object. Do not use a simplified `{ type, payload }` wrapper.

Key field clarifications:
- `path.operational_time_type`: Required; valid values are "24hours", "schedule", or "date_range"
- `path.channels[]`: Array of selected channel integrations; use Hub integration UUID as channel id
- `root_reply.name`: Required identifier for the root bot response; prefer a simple safe value like `Root` when upstream validation is unclear
- `root_reply.content.is_send_message`: Required boolean; should be true for user-facing messages
- `root_reply.content.channel_integration_id`: Must match the path-level `channel_integration_id`
- `root_reply.content.content_type_code`: Must be a valid code from `list_content_types`

Retry safety:
- `workflow_create_path_with_reply` is not guaranteed to be atomic
- if root reply initialization fails, the path may still have been created
- before retrying, check recent paths or use a path read to avoid creating duplicates
- successful create still leaves the Conversation in draft state; publish only when the task asks for it

Use `create_path` followed by `create_root_reply` only when the task explicitly needs those as separate steps.

## Update Path Or Conversation Metadata

Preferred tool:
- `update_path_metadata`

Use this when the user asks to update Conversation settings without editing a specific bot-response node.

Before the update:
1. Keep or re-resolve `changes.channel_integration_id` from `list_chatbot_channel_integrations`.
2. If the requested channel list came from Hub selections, normalize it through `normalize_hub_integrations_to_path_channels`.
3. Send the full merged metadata slice expected by the path update contract.

Backend behavior to remember:
- update revalidates submitted Hub channel ids
- update `find_or_create`s local integrations again
- update creates missing `path_channels` and deletes removed ones

## Add a Branch With a Reply

Preferred tool:
- `workflow_add_branch_with_reply`

Recommended request:

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "branch": {
    "user_input": {
      "input": "1",
      "channel_integration_id": 123,
      "is_default": false
    },
    "bot_response": {
      "name": "Menu 1 Reply",
      "content": {
        "content_text": "You selected menu 1",
        "content_type_code": "text",
        "channel_integration_id": 123,
        "is_send_message": true
      }
    }
  }
}
```

Recommended request using `add_branch` directly:

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "user_input": {
    "input": "Need help",
    "channel_integration_id": 123,
    "is_default": false
  },
  "bot_response": {
    "name": "Child Reply",
    "content": {
      "content_text": "How can I help?",
      "content_type_code": "text",
      "channel_integration_id": 123,
      "is_send_message": true
    }
  }
}
```

Behavior to expect:
- user input is created first
- child bot response is created second
- the tool returns both created nodes and the refreshed tree
- if the second step fails, the response should preserve created ids as partial-failure context
- ambiguous failures can still leave the user input or the full branch created

Critical required fields on `user_input`:
- `input`: The user-facing text or trigger keyword
- `channel_integration_id`: Required; the chatbot-local integer id
- `is_default`: Required boolean; set to true for exactly one default branch per parent, false for others

Note on `is_default`: The `add_branch` tool requires this field at creation time. If you need to toggle it later, use `set_default_user_input` separately.

Retry safety:
- do not blindly retry branch creation after an ambiguous error
- first re-read the parent bot response with `get_node_detail(path_id, "bot_response", parent_bot_response_id)`
- if that read still does not clearly show the new child, use `get_path_tree(path_id, preferred_tree_version="v3")` as the authoritative check
- v3 is the only tree route the backend serves; legacy v2/v1 tree reads now 404, so there is no v2 fallback to fall back to
- if a retry now fails with `Data user_input already exist with input = X`, treat that as evidence that an earlier call already created at least the branch trigger

## Create a Simple Non-Root Child Reply

Preferred tool:
- `create_bot_response`

Use this only when:
- the parent already exists
- the target node is not root
- the payload fits the validated child-create slice

If the target child needs advanced interactive or attachment behavior, create a simple child first and then update it through `update_bot_response`.

## Create or Update a WhatsApp Flow node

Preferred tools:
- `create_whatsapp_flow` to add a new WhatsApp Flow node
- `update_whatsapp_flow` to change an existing WhatsApp Flow node

Recommended `create_whatsapp_flow` request:

```json
{
  "path_id": 123,
  "whatsapp_flow": {
    "name": "Order Status Flow",
    "previous_bot_response_id": 456,
    "whatsapp_flow": {
      "template_id": "tmpl_123",
      "external_id": "ext_123",
      "template_name": "order_status_flow",
      "header_text": "Order Status",
      "message_content": "Tap below to check your order",
      "button_text": "Check status",
      "next_intent_type": "BRANCH"
    }
  }
}
```

Notes:
- the write goes to v1 (`POST /v1/bot_responses/whatsapp_flow`); the read-back tree is v3
- `channel_integration_id` is injected from the path metadata when omitted at the parent level
- the parent requires `name` and exactly one of `previous_bot_response_id` or `previous_user_input_id` (or a `parent` object `{ type, id }`)
- the nested `whatsapp_flow` requires `template_id`, `external_id`, `template_name`, `message_content`, `button_text` (<=60), and `next_intent_type` from {TEXT, BUTTON, LIST, AI_ASSIST, BRANCH}; `header_text` is optional
- use `update_whatsapp_flow(path_id, bot_response_id, changes)` to edit an existing flow node; read-back is via v3

## Change an Existing Node Component

Preferred tool:
- `workflow_change_node_component`

Fallback direct tool:
- `update_bot_response`

Recommended sequence:

1. Call `get_node_detail` for the target `bot_response` node.
2. Prepare `bot_response.content` and any component or sibling fields.
3. Call `workflow_change_node_component` with `transport="auto"` unless the task explicitly requires a transport restriction.
4. Check `meta.api_version_used` to see whether the server used v2 JSON, v3 JSON, or v3 multipart.
5. Inspect the refreshed node and tree from the response.

Use this for:
- interactive buttons
- interactive lists
- attachments
- API actions
- CRM deal payloads
- knowledge sources
- AI API knowledge
- conversation closure

## Update or Delete User Inputs

Recommended sequence:

1. Read the current branch via `get_node_detail(path_id, "user_input", user_input_id)`.
2. For ordinary field changes, call `update_user_input`.
3. For default toggling, call `set_default_user_input`.
4. For deletion, call `delete_user_input` with the narrowest acceptable `mode`.
5. Inspect the refreshed tree to confirm the branch outcome.

## Publish or Discard Draft Changes

### Publish

Use `publish_path` after one or more successful draft mutations.

Recommended request:

```json
{
  "path_id": 123,
  "strategy": "auto",
  "preferred_tree_version": "v3"
}
```

### Discard

Use `discard_path_draft` when the task explicitly asks to revert draft edits.

Recommended request:

```json
{
  "path_id": 123,
  "preferred_tree_version": "v3"
}
```

Both tools:
- return refreshed `path_metadata`
- return a refreshed `tree`
- surface fallback behavior through `warnings` and `meta`

## Safe Default Workflow Pattern

For most editing tasks, use this pattern:

1. Read path metadata.
2. Read the current tree.
3. Read the target node when the task is node-scoped.
4. Apply the narrowest fitting write tool.
5. Inspect `warnings` and `meta`.
6. Publish only if the task asks for publish or if the surrounding process requires it.

## Hub Lookup Workflows

Use these when FE flows read Hub state before or alongside chatbot editing.

### Auth bootstrap

Recommended order:

1. `get_hub_client_config`
2. `get_hub_user_profile`
3. `get_hub_organization_me`
4. `get_hub_billing_info`

Why this order is useful:
- it resolves client-config flags before later Hub calls depend on them
- it collects profile, organization, and billing state in a stable bootstrap sequence

### Conversation setup

Recommended order:

1. `list_hub_integrations`
2. choose the integration for the conversation flow
3. continue with chatbot path or conversation tools

Why this order is useful:
- it resolves Hub integration selection before path-level writes need `channels[]`

### Tag, division, and user lookup before component editing

Recommended order:

1. `list_hub_tags`
2. `list_hub_divisions`
3. `list_hub_users`
4. continue with chatbot node update tools

Why this order is useful:
- it loads the lookup data commonly needed by assignment, tagging, and routing edits before mutating a node

### Attachment preparation

Recommended order:

1. `upload_hub_message_file`
2. use the upload result in the chatbot bot-response update flow

Why this order is useful:
- it obtains upload metadata before the chatbot node update references the attachment

### Qontak CRM lookup

Recommended order:

1. `get_hub_qontak_integration_uniq`
2. continue with the CRM-related chatbot edit

Why this order is useful:
- it resolves CRM integration context before the chatbot node payload depends on it

### Division and user resolution by saved id

Recommended order:

1. `list_hub_divisions` or `list_hub_users` with the default first page
2. if the saved id is not present, retry with `payload.query=<id>` and a narrow limit

Why this order is useful:
- it keeps the first lookup cheap while still allowing exact-id recovery when saved labels are missing
