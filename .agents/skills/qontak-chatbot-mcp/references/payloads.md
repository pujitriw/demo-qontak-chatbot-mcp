# Payloads

## Shared Normalized Shapes

### `PathMetadata`

UI wording note:
- in the frontend, this is the Conversation metadata payload

```json
{
  "name": "Customer Service Main Path",
  "bot_type": "chat",
  "use_keyword": true,
  "keywords": ["customer service", "help"],
  "operational_time_type": "24hours",
  "channels": [
    {
      "id": "1",
      "name": "wa",
      "target_channel": "wa"
    }
  ],
  "schedules": [],
  "date_ranges": [],
  "channel_integration_id": 123,
  "nlp": {
    "enabled": true,
    "confidence_level": 0.85
  },
  "nlp_integration": {
    "id": 45
  }
}
```

Important:
- `channel_integration_id` is the chatbot-local owner id for the Conversation/path
- `channel_integration_id` must be a valid chatbot-local integer id, but it does not need to map 1:1 to every selected `channels[]` item
- `channels[]` is the selected Hub integration list attached to the Conversation/path
- request `channels[].id` is the selected Hub integration UUID
- response `channels[].id` is the chatbot-local `integrations.id`
- response `channels[].integration_id` is the original Hub integration UUID
- `operational_time_type` is a fixed CRUD enum with allowed values `24hours`, `schedule`, and `date_range`
- when `operational_time_type` is `date_range`, `date_ranges` must be non-empty; when it is `schedule`, `schedules` must be non-empty; `24hours` (shown here) requires neither, so leaving both empty is valid
- frontend path CRUD presents those values directly in the Conversation form and sends the chosen raw value in create and update payloads
- backend path CRUD validates the same enum inline; there is no separate list endpoint for these values
- `nlp.confidence_level` is a `0..1` float in the MCP (e.g. `0.85`, `0.9`), whereas the FE form shows a `1..100` percentage; convert the FE percent to a `0..1` fraction before sending (e.g. `90%` -> `0.9`), never send `90`

### Minimal `channels[]` item for path writes

```json
{
  "id": "fdfb5512-45d2-4f4e-b2fd-710f4176a17a",
  "name": "QontakSandbox5",
  "target_channel": "telegram"
}
```

Important for path writes:
- `id` must be the Hub integration UUID (string format, not numeric)
- `name` is the display name of the channel integration
- `target_channel` is the channel type (e.g., "telegram", "wa", "web_chat")
- do NOT include `integration_id` in the channels array for writes (response data includes it, but requests do not)
- if your source is a path read or `list_paths`, map response `channels[].integration_id` to request `channels[].id`

Use `normalize_hub_integrations_to_path_channels` when your source data comes from `list_hub_integrations` and still contains extra Hub-only fields.

### `ParentRef`

```json
{
  "type": "user_input",
  "id": 789
}
```

Allowed `type` values:
- `path`
- `bot_response`
- `user_input`

### `UserInputPayload`

```json
{
  "input": "Check order status",
  "channel_integration_id": 123,
  "is_default": false,
  "utterances": [
    {
      "text": "check my order",
      "organization_nlp_connector_id": 10
    }
  ],
  "organization_entity_id": 11,
  "organization_nlp_connector_id": 10
}
```

Required fields for `add_branch`:
- `input`: The user input text that triggers this branch
- `channel_integration_id`: The chatbot-local integer id (required for backend validation)
- `is_default`: Boolean flag indicating if this is the default branch (required; use `set_default_user_input` to toggle later)

Optional fields:
- `utterances`: Alternative utterance patterns for NLP matching
- `organization_entity_id`: Entity reference when using NLP
- `organization_nlp_connector_id`: NLP connector reference

Note: When using `update_user_input`, do NOT pass `is_default` (use `set_default_user_input` instead). When using `add_branch`, `is_default` is required at creation time.

### `BranchWithReplyPayload`

```json
{
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
```

Use this wrapper for `workflow_add_branch_with_reply`.

Important:
- `branch.user_input` and `branch.bot_response` must both be objects
- `branch.user_input` uses the same required fields as direct `add_branch`
- numbered menu choices like `1`, `2`, `3`, `4` are ordinary `input` strings; there is no special numbered-menu payload shape

### `BotResponsePayload`

```json
{
  "name": "Send order status menu",
  "content": {
    "content_text": "Choose one option",
    "content_type_code": "button",
    "is_send_message": true,
    "channel_integration_id": 123
  },
  "components": {
    "assignment": {
      "enabled": false
    },
    "auto_resolve": {
      "enabled": false
    },
    "idle_rule": {
      "enabled": false
    },
    "api": {
      "enabled": false
    }
  },
  "interactive": {
    "button": {
      "format": "text",
      "body_text": "Choose one option",
      "actions": [
        { "title": "Check status", "action_status": "CREATE" }
      ]
    },
    "list": null
  },
  "crm_deal": {
    "enabled": true,
    "is_contact_association": true,
    "api_body": {
      "name": "New inbound deal",
      "crm_pipeline_id": "pipe-1",
      "crm_stage_id": "stage-1",
      "creator_id": "user-9",
      "crm_source_id": "src-1"
    }
  },
  "tag": ["support"],
  "is_purchase_event": false,
  "content_type_version": "2.1",
  "organization_entity_id": 99
}
```

Note:
- `content_type_version` should be treated as an opaque value and passed through as provided
- on CHILD create (`create_bot_response`) the top-level `interactive`, `attachments`, `crm_deal`, `tag`, `is_purchase_event`, and `organization_entity_id` families are now carried through (previously dropped); a brand-new interactive is created inline via the v1 create path
- to REUSE an existing bot response instead of authoring content, send `"bot_response": { "id": <int>, "is_reuse": true }` on `create_bot_response` rather than a `content` block

Interactive validation (enforced, matches the chatbot dry-schema):
- supply exactly one of `interactive.button` or `interactive.list`
- `button.format` is required and must be one of `text`, `document`, `image`, `video`
- `button.body_text` <= 1024; `button.header_text` <= 60
- `button.actions` must hold 1 to 3 actions excluding any with `action_status="DELETE"`, so a non-empty actions list is required (not an empty array); each action `title` <= 20
- optional `button.attachment` is `{ channel_attachment_id (string), name, type, url }` with `type` in {document, image, video}
- `list.body_text` (required) <= 1024; `list.button_text` (required) <= 20; `list.header_text` <= 60
- `list.sections` >= 1 and each section `items` >= 1; each item `title` (required) <= 24 and `description` <= 72

### `WhatsAppFlowPayload`

```json
{
  "name": "Order Status Flow",
  "channel_integration_id": 123,
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
```

Use this shape for `create_whatsapp_flow`.

Important:
- parent level requires `name`, and exactly one of `previous_bot_response_id` or `previous_user_input_id` (a `parent` object `{ type, id }` is also accepted)
- `channel_integration_id` is injected from the path metadata when omitted
- nested `whatsapp_flow` requires `template_id`, `external_id`, `template_name`, `message_content`, `button_text` (<=60), and `next_intent_type` from {TEXT, BUTTON, LIST, AI_ASSIST, BRANCH}; `header_text` is optional
- the write targets v1 (`POST /v1/bot_responses/whatsapp_flow`); the read-back tree is v3

WhatsApp Flow update:
- use `update_whatsapp_flow(path_id, bot_response_id, changes)`; all nested `whatsapp_flow` fields are optional in the `changes` slice (patch only what you need), and read-back is via v3

### `AiAssistPayload`

```json
{
  "name": "Knowledge Reply",
  "content": {
    "content_text": "Let me look that up for you",
    "content_type_code": "ai_assist",
    "channel_integration_id": 123,
    "is_send_message": true
  },
  "ai_api_knowledge": {
    "enabled": true,
    "sources": [
      {
        "id": 1,
        "api_connection_id": 20
      }
    ]
  },
  "knowledge_sources": [
    {
      "id": 1
    }
  ]
}
```

Use this shape for `create_bot_response` with an `ai_assist` content type.

Important:
- `content.content_type_code` must be `ai_assist`
- `ai_api_knowledge` and `knowledge_sources` are TOP-LEVEL siblings of `content`/`components` for child create
- for the seeded root node (`create_root_reply`), nest those same families INSIDE `components` instead
- the write targets v1 (`POST /v1/bot_responses`, ai_assist branch); the knowledge is wired during create

### `ExitConditionPayload`

```json
{
  "name": "Escalate to agent",
  "description": "Hand off when the model cannot answer",
  "trigger": "fallback",
  "next_intent_type": "AI_AGENT",
  "ai_agent_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a"
}
```

Use this shape as the `exit_condition` argument for `create_exit_condition`.

Important:
- `next_intent_type` is one of `TEXT`, `BUTTON`, `LIST`, `BRANCH`, `AI_AGENT`, `WHATSAPP_FLOW` (auto-uppercased)
- `ai_agent_id` is required only when `next_intent_type=AI_AGENT`
- `name`, `description`, and `trigger` are optional
- creates a user input with `response_type="exit_condition"` via `POST /v2/user_inputs`; the tool auto-sets `parameters.path_id`

### `AiAgentPayload`

```json
{
  "name": "Sales Assistant Node",
  "contentable_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a",
  "content_type_version": 1,
  "channel_integration_id": 123,
  "previous_bot_response_id": 456
}
```

Use this shape as the `ai_agent` argument for `create_ai_agent_response`.

Important:
- `name` (unique), `contentable_id` (the agent UUID from `list_ai_agents`), and `content_type_version` are required
- `channel_integration_id` is injected from path metadata when omitted
- supply exactly one of `previous_bot_response_id` / `previous_user_input_id` (a `parent` object `{ type, id }` is also accepted)
- the agent must pre-exist; `contentable_type` is fixed to `ai_agent` server-side
- the write targets v2 (`POST /v2/bot_responses`); the node is resolved by name diff since the endpoint returns no id

### `BranchConditionPayload`

```json
{
  "name": "Order status router",
  "channel_integration_id": 123,
  "previous_bot_response_id": 456,
  "api": {
    "enabled": true,
    "connection_id": 20,
    "path": "/orders/status",
    "method": "GET",
    "success_key": "status",
    "success_value": "ok",
    "response_conditions": [
      {
        "sequence": 1,
        "next_intent_type": "TEXT",
        "is_default": false,
        "criterias": [
          {
            "key": "status",
            "operator": "equals",
            "value": "shipped",
            "sequence": 1,
            "condition_operator": "and"
          }
        ]
      }
    ]
  }
}
```

Use this shape as the `branch` argument for `create_branch_condition`.

Important:
- this is the CONDITION branch (content type `branch`), DISTINCT from `add_branch` (a conversational user-input menu branch)
- `name` (unique) is required; `channel_integration_id` is injected when omitted
- supply exactly one of `previous_bot_response_id` / `previous_user_input_id` (a `parent` object `{ type, id }` is also accepted)
- optional `id` re-creates an existing intent as a branch, destroying its children
- in the `api` block, `enabled` is required; when `enabled=true`, `connection_id`, `path`, and `method` (one of `GET`/`POST`/`PUT`/`PATCH`/`DELETE`/`COPY`/`HEAD`/`OPTIONS`) are required
- each `response_conditions[]` needs `sequence` (int) and `next_intent_type` (`TEXT`/`BUTTON`/`LIST`/`AI_ASSIST`, required on create); the backend auto-creates each condition's downstream node
- each `criterias[]` needs `key`, `operator`, and `sequence` (int); `value` and `condition_operator` are optional
- the write targets v1 (`POST /v1/bot_responses/branches`); the node is resolved by name diff

### `BranchConditionUpdatePayload`

```json
{
  "name": "Order status router",
  "channel_integration_id": 123,
  "api": {
    "enabled": true,
    "connection_id": 20,
    "path": "/orders/status",
    "method": "GET",
    "response_conditions": [
      {
        "id": 5001,
        "action_status": "update",
        "sequence": 1,
        "is_default": false,
        "criterias": [
          {
            "id": 6001,
            "action_status": "update",
            "key": "status",
            "operator": "equals",
            "value": "delivered",
            "sequence": 1
          }
        ]
      }
    ]
  }
}
```

Use this shape as the `changes` argument for `update_branch_condition`.

Important:
- `name` is required; `channel_integration_id` is injected when omitted
- the `api` / `response_conditions` / `criterias` structure matches `create_branch_condition`, EXCEPT `response_conditions[]` and `criterias[]` may additionally carry `id` and `action_status` (`create` / `update` / `delete`) for diff-style edits of existing rows
- `next_intent_type` is not enforced on update
- the write targets v1 (`PATCH /v1/bot_responses/branches/{bot_response_id}`)

### `UserInputPayload` (OCR / multitype)

```json
{
  "input": "Upload your receipt",
  "is_default": false,
  "response_type": "image",
  "ai_knowledge_source_ocr_enabled": true,
  "ai_knowledge_source_ocr_id": 42,
  "utterances": [
    { "text": "here is my receipt" }
  ]
}
```

Use this shape as the `user_input` argument for `create_user_input`.

Important:
- `response_type` is one of `text`, `image`, `all_types`; OCR / multitype matching uses `image` or `all_types`
- `ai_knowledge_source_ocr_id` is an `ai_knowledge_source_id` (not a node id)
- creates the user input only (no child reply) via `POST /v2/user_inputs`; attach a reply later with `create_bot_response`

### `FallbackPayload`

```json
{
  "bot_response": {
    "name": "Default Fallback",
    "content_type_id": 1,
    "content_text": "Sorry, I did not understand that.",
    "intent_type": "FALLBACK",
    "is_send_message": true,
    "assignment": {
      "enabled": false
    }
  }
}
```

Use this shape as the `fallback` argument for `create_fallback`.

Important:
- `intent_type` is `FALLBACK` for a plain default reply or `AI_ASSIST` for a knowledge-backed fallback
- the write targets v1 (`POST /v1/user_inputs`)

### `FallbackUpdatePayload`

```json
{
  "content_text": "Sorry, I still did not understand. Let me connect you to an agent.",
  "is_send_message": true,
  "assignment": {
    "enabled": true,
    "is_auto": true,
    "division": {
      "channel_division_id": "division-1"
    }
  }
}
```

Use this shape as the `changes` argument for `update_fallback`.

Important:
- `assignment` is REQUIRED by the backend; include at least `{ "enabled": false }`
- `channel_integration_id` is injected by the tool
- the write targets v1 (`PATCH /v1/bot_responses/{bot_response_id}/fallback`)

### `AiAgentUpdatePayload`

```json
{
  "name": "Sales Assistant Node",
  "contentable_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a",
  "content_type_version": 1
}
```

Use this shape as the `changes` argument for `update_ai_agent_response`.

Important:
- `contentable_type` is fixed to `ai_agent` server-side; `path_id` and `parameters` are optional
- the write targets v3 (`PATCH /v3/bot-responses/{bot_response_id}`)

### `BranchConditionPayload` (typed `conditions[]`, v2)

A top-level `conditions[]` array adds SCHEDULE / CHANNEL / CUSTOMER_FIELD branch types and auto-routes the write to the v2 branches endpoint.

CUSTOMER_FIELD with `is_between`:

```json
{
  "name": "Age range router",
  "channel_integration_id": 123,
  "previous_bot_response_id": 456,
  "conditions": [
    {
      "branch_type": "CUSTOMER_FIELD",
      "sequence": 1,
      "next_intent_type": "TEXT",
      "parameters": {
        "key": "age",
        "operator": "is_between",
        "value": { "min": 18, "max": 30 }
      }
    }
  ]
}
```

SCHEDULE branch:

```json
{
  "name": "Office hours router",
  "channel_integration_id": 123,
  "previous_bot_response_id": 456,
  "conditions": [
    {
      "branch_type": "SCHEDULE",
      "sequence": 1,
      "next_intent_type": "TEXT",
      "schedules": [
        {
          "day_number": 1,
          "start_hour": "09:00:00",
          "end_hour": "17:00:00",
          "is_24h": false,
          "cross_midnight": false,
          "sequence": 1,
          "condition_operator": "and"
        }
      ]
    }
  ]
}
```

Use these shapes as the `branch` argument for `create_branch_condition` (or `changes` for `update_branch_condition`).

Important:
- each entry carries a `branch_type` plus its typed sub-object: `api { key, value }`, `schedules[]`, `channel { channel_id, channel_name, channel_type, operator, sequence }`, or `parameters {}` for CUSTOMER_FIELD
- CUSTOMER_FIELD `value` is a scalar for ordinary operators, or `{ min, max }` when the operator is `is_between`
- supplying `conditions[]` auto-routes to the v2 branches endpoint (`meta.api_version_used = "v2"`); the legacy v1 `api.response_conditions` shape still works

### `CrmDealUpdatePayload`

```json
{
  "enabled": true,
  "is_contact_association": true,
  "api_body": {
    "name": "New inbound deal",
    "crm_pipeline_id": "pipe-1",
    "crm_stage_id": "stage-1",
    "creator_id": "user-9",
    "crm_source_id": "src-1"
  }
}
```

Use this shape as the `changes` argument for `update_crm_deal`.

Important:
- pass `crm_deal_id` (the existing deal block id) as a separate tool argument
- `creator_id` / `creator_name` identifies the deal owner; `*_id` and `*_name` variants are interchangeable per field
- the write targets v1 (`PATCH /v1/bot_responses/{bot_response_id}/crm_deals/{crm_deal_id}`)

### `PathKnowledgeSourcesPayload`

```json
{
  "knowledge_sources": [
    { "id": 1 },
    { "id": 2 }
  ]
}
```

Use this as the `knowledge_sources` argument for `create_path_knowledge_sources`.

Important:
- this binds knowledge at the PATH level (`POST /v1/paths/{path_id}/knowledge-sources`), distinct from node-level ai_assist `knowledge_sources`

## Starter Requests

### Create a path only

UI wording note:
- use this shape when the user asks to create a Conversation without asking for the first reply in the same step

```json
{
  "path": {
    "name": "Customer Service Main Path",
    "bot_type": "chat",
    "use_keyword": true,
    "keywords": ["customer service", "help"],
    "operational_time_type": "24hours",
    "channels": [
      {
        "id": "fdfb5512-45d2-4f4e-b2fd-710f4176a17a",
        "name": "QontakSandbox5",
        "target_channel": "telegram"
      }
    ],
    "schedules": [],
    "date_ranges": [],
    "channel_integration_id": 123,
    "nlp": {
      "enabled": true,
      "confidence_level": 0.85
    },
    "nlp_integration": {
      "id": 45
    }
  }
}
```

Notes:
- `operational_time_type` is effectively required in create flows; `24hours` is the safest default when no schedule is requested
- if a channel is selected, include the normalized `channels[]` entry during create instead of relying only on `channel_integration_id`
- if Hub lookup tools are unavailable, recover the channel UUID from a trusted path read and place it in `channels[].id`

### Update path metadata

```json
{
  "path_id": 123,
  "changes": {
    "channel_integration_id": 123,
    "name": "Customer Service Main Path v2",
    "keywords": ["help", "support"],
    "use_keyword": true,
    "operational_time_type": "date_range",
    "channels": [],
    "schedules": [],
    "date_ranges": []
  }
}
```

Important:
- always include `changes.channel_integration_id`
- `changes.channel_integration_id` must be valid for the chatbot org, but it does not need to map 1:1 to the selected `changes.channels[]` items
- `changes.channels` uses the same minimal Hub selection payload shape as create
- this is also the right shape when the user asks to update Conversation settings in the UI
- keep using the fixed `operational_time_type` values `24hours`, `schedule`, or `date_range`
- if `operational_time_type` becomes `schedule`, `schedules` must not be empty
- when `operational_time_type` is `schedule`, every schedule item must use `day_number` in `0..6`, `start_hour` and `end_hour` in `HH:MM:SS`, and `start_hour < end_hour`
- if `operational_time_type` becomes `date_range`, `date_ranges` must not be empty
- when `operational_time_type` is `date_range`, every date range item must use valid datetime strings and `start_date < end_date`

### Initialize a root reply

```json
{
  "path_id": 123,
  "bot_response": {
    "name": "Root",
    "content": {
      "content_text": "Welcome to support",
      "content_type_code": "text",
      "is_send_message": true,
      "channel_integration_id": 123,
      "content_type_version": "1.0"
    },
    "tag": ["support", "root"],
    "components": {
      "assignment": {
        "enabled": false
      }
    }
  }
}
```

Notes:
- `name` is required on root initialization; use a simple safe identifier like `Root` if upstream validation is unclear
- include `is_send_message: true` for ordinary text replies shown to end users

### Add a branch with the next reply

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "user_input": {
    "input": "Check order status",
    "channel_integration_id": 123,
    "is_default": false,
    "utterances": [
      {
        "text": "check my order",
        "organization_nlp_connector_id": 10
      }
    ]
  },
  "bot_response": {
    "name": "Order Status Menu",
    "content": {
      "content_text": "Here is the answer",
      "content_type_code": "text",
      "channel_integration_id": 123,
      "is_send_message": true
    },
    "components": {
      "assignment": {
        "enabled": false
      }
    }
  }
}
```

### Create a simple non-root child bot response

```json
{
  "path_id": 123,
  "parent": {
    "type": "user_input",
    "id": 789
  },
  "bot_response": {
    "name": "Connect To Agent",
    "content": {
      "content_text": "We will connect you to an agent",
      "content_type_code": "text",
      "channel_integration_id": 123,
      "is_send_message": true
    },
    "components": {
      "assignment": {
        "enabled": true,
        "is_auto": true,
        "division": {
          "channel_division_id": "division-1"
        }
      }
    }
  }
}
```

### Update an existing bot response

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "transport": "auto",
  "bot_response": {
    "name": "Order Status Menu",
    "content_type_version": "2.1",
    "content": {
      "content_text": "Choose one option",
      "content_type_code": "button",
      "channel_integration_id": 123,
      "is_send_message": true
    },
    "interactive": {
      "button": {
        "format": "text",
        "header_text": "Order Menu",
        "body_text": "Choose one option",
        "actions": [
          { "title": "Check status", "action_status": "CREATE" }
        ]
      }
    }
  }
}
```

### Create an ai_assist child reply

```json
{
  "path_id": 123,
  "parent": {
    "type": "user_input",
    "id": 789
  },
  "bot_response": {
    "name": "Knowledge Reply",
    "content": {
      "content_text": "Let me look that up for you",
      "content_type_code": "ai_assist",
      "channel_integration_id": 123,
      "is_send_message": true
    },
    "ai_api_knowledge": {
      "enabled": true,
      "sources": [
        {
          "id": 1,
          "api_connection_id": 20
        }
      ]
    },
    "knowledge_sources": [
      {
        "id": 1
      }
    ]
  }
}
```

Notes:
- `ai_api_knowledge` and `knowledge_sources` sit at the top level of `bot_response` (siblings of `content`/`components`) for child create
- for the seeded root node, move both families inside `components` and use `create_root_reply`

### Create an exit condition

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "is_default": false,
  "exit_condition": {
    "name": "Escalate to agent",
    "next_intent_type": "AI_AGENT",
    "ai_agent_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a"
  }
}
```

Notes:
- `next_intent_type` is auto-uppercased; `ai_agent_id` is required only for `AI_AGENT`
- the tool sets `parameters.path_id` itself; do not pass it inside `exit_condition`

### Create an ai_agent response node

```json
{
  "path_id": 123,
  "ai_agent": {
    "name": "Sales Assistant Node",
    "contentable_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a",
    "content_type_version": 1,
    "channel_integration_id": 123,
    "previous_bot_response_id": 456
  }
}
```

Notes:
- resolve `contentable_id` with `list_ai_agents` first; the agent must already exist
- pass exactly one of `previous_bot_response_id` / `previous_user_input_id`, or a `parent` object `{ type, id }`

### Create a condition branch

```json
{
  "path_id": 123,
  "branch": {
    "name": "Order status router",
    "channel_integration_id": 123,
    "previous_bot_response_id": 456,
    "api": {
      "enabled": true,
      "connection_id": 20,
      "path": "/orders/status",
      "method": "GET",
      "response_conditions": [
        {
          "sequence": 1,
          "next_intent_type": "TEXT",
          "is_default": false,
          "criterias": [
            {
              "key": "status",
              "operator": "equals",
              "value": "shipped",
              "sequence": 1
            }
          ]
        }
      ]
    }
  }
}
```

Notes:
- this is the CONDITION branch, DISTINCT from `add_branch` (a conversational menu branch)
- when `api.enabled=true`, `connection_id`, `path`, and `method` are required; each condition needs `sequence` and (on create) `next_intent_type`

### Update a condition branch

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "changes": {
    "name": "Order status router",
    "channel_integration_id": 123,
    "api": {
      "enabled": true,
      "connection_id": 20,
      "path": "/orders/status",
      "method": "GET",
      "response_conditions": [
        {
          "id": 5001,
          "action_status": "update",
          "sequence": 1,
          "criterias": [
            {
              "id": 6001,
              "action_status": "update",
              "key": "status",
              "operator": "equals",
              "value": "delivered",
              "sequence": 1
            }
          ]
        }
      ]
    }
  }
}
```

Notes:
- conditions and criterias may carry `id` + `action_status` (`create`/`update`/`delete`) for diff-style edits
- `next_intent_type` is not enforced on update

### Configure a voice reply (conversation closure + audio attachment)

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "transport": "auto",
  "bot_response": {
    "name": "Voice Reply",
    "content": {
      "content_text": "Thanks for calling",
      "content_type_code": "text",
      "channel_integration_id": 123,
      "is_send_message": true
    },
    "conversation_closure": {
      "enabled": true,
      "action_type": "resolve",
      "resolve": {
        "text_message": "Conversation closed"
      }
    },
    "attachments": [
      {
        "name": "greeting.mp3",
        "type": "audio",
        "upload": {
          "file_path": "/absolute/path/to/greeting.mp3"
        }
      }
    ]
  }
}
```

Notes:
- voice is not a separate node type; configure it on a bot response via `update_bot_response` (auto-routes to v3 for `conversation_closure` and file-backed attachments)
- the FE `action` / call-group field (assign_to_call_group / end_the_call) has no backend counterpart and is omitted
- `bot_type="voice"` remains a path-level setting, not part of this node payload

### Reuse an existing bot response on child create

```json
{
  "path_id": 123,
  "parent": {
    "type": "user_input",
    "id": 789
  },
  "bot_response": {
    "id": 456,
    "is_reuse": true
  }
}
```

Notes:
- the child links to the existing bot response `456` instead of authoring new content
- omit `content` when reusing

### Create a standalone user input (OCR / multitype)

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "user_input": {
    "input": "Upload your receipt",
    "is_default": false,
    "response_type": "image",
    "ai_knowledge_source_ocr_enabled": true,
    "ai_knowledge_source_ocr_id": 42
  }
}
```

Notes:
- creates the user input only (no child reply); attach a reply later with `create_bot_response`
- `ai_knowledge_source_ocr_id` is an `ai_knowledge_source_id`

### Create a default fallback

```json
{
  "path_id": 123,
  "parent_bot_response_id": 456,
  "fallback": {
    "bot_response": {
      "name": "Default Fallback",
      "content_type_id": 1,
      "content_text": "Sorry, I did not understand that.",
      "intent_type": "FALLBACK",
      "is_send_message": true,
      "assignment": {
        "enabled": false
      }
    }
  }
}
```

Notes:
- `intent_type` is `FALLBACK` or `AI_ASSIST`
- the write targets v1 (`POST /v1/user_inputs`)

### Update a fallback

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "changes": {
    "content_text": "Let me connect you to an agent.",
    "is_send_message": true,
    "assignment": {
      "enabled": false
    }
  }
}
```

Notes:
- `assignment` is REQUIRED by the backend; include at least `{ "enabled": false }`
- the write targets v1 (`PATCH /v1/bot_responses/{id}/fallback`)

### Update an ai_agent response node

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "changes": {
    "name": "Sales Assistant Node",
    "contentable_id": "0b8c2c8e-1f4a-4f1d-9c2a-2b6c0d3e4f5a",
    "content_type_version": 1
  }
}
```

Notes:
- `contentable_type` is fixed to `ai_agent` server-side
- the write targets v3 (`PATCH /v3/bot-responses/{id}`)

### Create a CUSTOMER_FIELD `is_between` condition branch (v2)

```json
{
  "path_id": 123,
  "branch": {
    "name": "Age range router",
    "channel_integration_id": 123,
    "previous_bot_response_id": 456,
    "conditions": [
      {
        "branch_type": "CUSTOMER_FIELD",
        "sequence": 1,
        "next_intent_type": "TEXT",
        "parameters": {
          "key": "age",
          "operator": "is_between",
          "value": { "min": 18, "max": 30 }
        }
      }
    ]
  }
}
```

Notes:
- a top-level `conditions[]` auto-routes to the v2 branches endpoint
- for `is_between`, `value` is `{ min, max }`; other operators use a scalar `value`

### Create a SCHEDULE condition branch (v2)

```json
{
  "path_id": 123,
  "branch": {
    "name": "Office hours router",
    "channel_integration_id": 123,
    "previous_bot_response_id": 456,
    "conditions": [
      {
        "branch_type": "SCHEDULE",
        "sequence": 1,
        "next_intent_type": "TEXT",
        "schedules": [
          {
            "day_number": 1,
            "start_hour": "09:00:00",
            "end_hour": "17:00:00",
            "is_24h": false,
            "cross_midnight": false,
            "sequence": 1,
            "condition_operator": "and"
          }
        ]
      }
    ]
  }
}
```

Notes:
- a top-level `conditions[]` auto-routes to the v2 branches endpoint

### Update a CRM deal

```json
{
  "path_id": 123,
  "bot_response_id": 456,
  "crm_deal_id": 77,
  "changes": {
    "enabled": true,
    "is_contact_association": true,
    "api_body": {
      "name": "New inbound deal",
      "crm_pipeline_id": "pipe-1",
      "crm_stage_id": "stage-1",
      "creator_id": "user-9",
      "crm_source_id": "src-1"
    }
  }
}
```

Notes:
- `creator_id` / `creator_name` is the deal owner
- the write targets v1 (`PATCH /v1/bot_responses/{id}/crm_deals/{crm_id}`)

### Attach path-level knowledge sources

```json
{
  "path_id": 123,
  "knowledge_sources": [
    { "id": 1 },
    { "id": 2 }
  ]
}
```

Notes:
- PATH-LEVEL knowledge (`POST /v1/paths/{id}/knowledge-sources`), distinct from node-level ai_assist `knowledge_sources`

## Payload Constraints

### Path updates

- `channel_integration_id` is mandatory for `update_path_metadata`
- keep merged operational-time payloads valid

### Child bot response creation

- use `create_bot_response` for simple child creation
- keep it to the validated child-create slice based on text content plus JSON-safe component groups
- use `update_bot_response` after creation if the node needs advanced components

### Root reply initialization

- use `create_root_reply` only for the seeded root node returned by the path tree
- do not use it to create a second root

### Node updates

- use `update_bot_response` for existing-node component edits
- inspect `meta.api_version_used` to confirm whether the request stayed on v2 JSON or routed to v3

## Hub Payloads

Hub request shapes should follow the documented MCP defaults in this skill bundle.

### Get the current Hub user profile

```json
{}
```

### List Hub users

```json
{
  "payload": {
    "offset": 1,
    "limit": 10
  }
}
```

Use this as the default payload when the caller does not specify other pagination values.

### Get the current Hub organization

```json
{}
```

Note:
- some clients may add tag-style params to this call
- treat that as a caller quirk rather than a required semantic default

### List Hub integrations

```json
{
  "payload": {
    "offset": 1,
    "limit": 100,
    "query": ""
  }
}
```

Notes:
- keep the MCP-facing request on the flat pagination values shown here
- if a caller produces a nested pagination shape, normalize it before sending

### List Hub divisions

```json
{
  "payload": {
    "offset": 1,
    "limit": 100,
    "query": ""
  }
}
```

Use this as the default payload when the caller does not specify other pagination values.

### List Hub tags

```json
{
  "payload": {
    "offset": 1,
    "limit": 100,
    "query": ""
  }
}
```

Use this as the default payload when the caller does not specify other pagination values.

### Get Hub billing info

```json
{}
```

### Get Hub client config

```json
{}
```

### Upload a Hub message file

```json
{
  "file_name": "guide.pdf",
  "content_type": "application/pdf",
  "content_bytes_base64": "<base64>",
  "payload": {
    "source": "attachment-form"
  }
}
```

Notes:
- FE sends an opaque body to `$apiHub`
- MCP should expose file content explicitly and translate it into upstream multipart form data

### Get Hub Qontak integration uniq

```json
{
  "payload": {
    "integration_id": "<integration-uuid>"
  }
}
```

Notes:
- `payload` is required (it has no default) and is passed through as query params
- the exact keys depend on the caller's lookup; `integration_id` is shown as a concrete example, not a fixed contract
- pass through only caller-supplied params

### Get Hub billing info

```json
{
  "payload": {}
}
```

Notes:
- no FE default payload is confirmed

### Get Hub client config

```json
{}
```

Notes:
- FE forces staging for this call via `$apiHub(..., true)`
- do not assume that staging override applies to the rest of the Hub tool family
