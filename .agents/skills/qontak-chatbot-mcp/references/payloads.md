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
  "operational_time_type": "date_range",
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
- frontend path CRUD presents those values directly in the Conversation form and sends the chosen raw value in create and update payloads
- backend path CRUD validates the same enum inline; there is no separate list endpoint for these values

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
      "actions": []
    },
    "list": null
  },
  "tag": ["support"],
  "content_type_version": "2.1",
  "organization_entity_id": 99
}
```

Note:
- `content_type_version` should be treated as an opaque value and passed through as provided

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
    "content": {
      "content_text": "Choose one option",
      "content_type_code": "button",
      "channel_integration_id": 123,
      "is_send_message": true,
      "content_type_version": "2.1"
    },
    "interactive": {
      "button": {
        "format": "text",
        "header_text": "Order Menu",
        "body_text": "Choose one option",
        "actions": []
      }
    }
  }
}
```

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
    "limit": 10,
    "query": ""
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
  "metadata": {
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
    "integration_id": "optional"
  }
}
```

Notes:
- no FE default payload is confirmed
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
