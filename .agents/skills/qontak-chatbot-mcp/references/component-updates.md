# Component Updates

## Purpose

This reference covers the component families and transport rules that matter when editing existing bot response nodes.

The central rule is simple:
- use `update_bot_response` or `workflow_change_node_component` for advanced component edits
- do not overload `create_bot_response` with component families that are only validated on the existing-node update path

## Transport Selection

### `transport="auto"`

Default choice for most edits.

Behavior:
- keeps ordinary JSON-safe edits on v2 JSON
- routes `conversation_closure` to v3 JSON
- routes file-backed attachment uploads to v3 multipart

### `transport="json"`

Use only when the task must stay on a JSON-only path.

Constraint:
- this must reject multipart-only payloads such as file-backed attachments or conversation closure

### `transport="multipart"`

Use when the task explicitly requires multipart, typically because at least one attachment carries `upload.file_path`.

## Supported Component Families On Existing-Node Update

The validated FE-grounded update slice supports:
- `tag`
- `interactive`
- `attachments`
- `knowledge_sources`
- `ai_api_knowledge`
- `components.api`
- `components.assignment`
- `components.auto_resolve`
- `components.idle_rule`
- `components.crm_deal`
- `components.conversation_closure`
- `content_type_version`
- `contentable_type`
- `contentable_id`
- `organization_entity_id`
- `path_id`

## Supported Component Families On Child Create

The validated child-create slice is intentionally narrower.

Use `create_bot_response` for:
- text content
- `components.assignment`
- `components.auto_resolve`
- `components.idle_rule`
- `components.api`

Do not rely on child-create for:
- interactive button or list payloads
- attachments
- CRM payloads
- knowledge-source payloads
- conversation closure

For those, create the node first if needed, then update it.

## Component Examples

### API action

```json
{
  "components": {
    "api": {
      "enabled": true,
      "connection_id": 123,
      "path": "/orders/status",
      "method": "GET",
      "headers": "{\"X-Token\":\"abc\"}",
      "body": "{}",
      "success_key": "status",
      "success_value": "ok",
      "fallback_bot_response_id": 999,
      "is_api_entity": true,
      "entities": [
        {
          "entity": {
            "id": 1
          },
          "action_status": "UPDATE"
        }
      ]
    }
  }
}
```

### Assignment

```json
{
  "components": {
    "assignment": {
      "enabled": true,
      "is_auto": true,
      "division": {
        "channel_division_id": "division-1"
      },
      "agent": {
        "channel_agent_id": "agent-1"
      }
    }
  }
}
```

### Interactive button

```json
{
  "interactive": {
    "button": {
      "format": "text",
      "header_text": "Header",
      "body_text": "Choose one option",
      "attachment": {
        "channel_attachment_id": "att-1",
        "name": "sample.png",
        "type": "image",
        "url": "https://example.com/image.png"
      },
      "actions": [
        {
          "code": "CHECK_STATUS",
          "title": "Check status",
          "sequence": 1,
          "action_status": "CREATE"
        }
      ]
    }
  }
}
```

Note:
- the request contract currently supports one `attachment` object inside the interactive button payload, not an attachment array

### Interactive list

```json
{
  "interactive": {
    "list": {
      "is_merge": false,
      "format": "text",
      "header_text": "Header",
      "body_text": "Choose one option",
      "button_text": "Open menu",
      "sections": [
        {
          "sequence": 1,
          "items": [
            {
              "code": "CHECK_STATUS",
              "title": "Check status",
              "description": "Track current order",
              "sequence": 1,
              "response_id": 1001,
              "action_status": "CREATE"
            }
          ]
        }
      ]
    }
  }
}
```

### Interactive validation rules

Interactive payloads are validated client-side to match the chatbot dry-schema. The constraints are enforced before the write is sent:

- exactly one of `button` or `list` must be present; a cleared/no-interactive value is `{"button": null, "list": null}` or an empty object
- button:
  - `format` is required and must be one of `text`, `document`, `image`, `video`
  - `header_text` optional, max 60 chars
  - `body_text` required, max 1024 chars
  - `actions` must contain 1 to 3 actions excluding any with `action_status="DELETE"` (an empty `actions` array is invalid); each action `title` max 20 chars
  - optional `attachment`: `channel_attachment_id` (string), `name`, `type` one of `document`, `image`, `video`, and `url`
- list:
  - `header_text` optional, max 60 chars
  - `body_text` required, max 1024 chars
  - `button_text` required, max 20 chars
  - `sections` requires at least 1 section; each section `items` requires at least 1 item
  - each item `title` required, max 24 chars; item `description` optional, max 72 chars

### Creating vs updating interactive

- a brand-new interactive (`button` or `list`) is created inline only through the v1 create path (`create_bot_response`)
- `update_bot_response` and `create_root_reply` use the v2 write, which only updates an interactive that already exists, matched by `id` — they cannot create a brand-new interactive from scratch

### Attachments

```json
{
  "attachments": [
    {
      "id": 1,
      "channel_attachment_id": 10,
      "url": "https://example.com/file.pdf",
      "name": "guide.pdf",
      "type": "document",
      "caption": "Guide",
      "action_status": "UPDATE",
      "upload": {
        "file_path": "/absolute/path/to/guide.pdf"
      }
    }
  ]
}
```

Rules:
- use `upload.file_path` when the attachment must be uploaded through multipart
- if `upload.file_path` is absent, URL-only or existing-asset attachment edits may remain on JSON-safe transport

### CRM deal

```json
{
  "components": {
    "crm_deal": {
      "enabled": true,
      "is_contact_association": true,
      "api_body": {
        "name": "Support Case",
        "crm_pipeline_id": 1,
        "crm_pipeline_name": "Support",
        "crm_stage_id": 2,
        "crm_stage_name": "Open",
        "creator_id": 3,
        "creator_name": "Bot",
        "crm_source_id": 4,
        "crm_source_name": "Chatbot"
      }
    }
  }
}
```

### Knowledge sources

```json
{
  "knowledge_sources": [
    {
      "id": 1
    }
  ]
}
```

### AI API knowledge

```json
{
  "components": {
    "ai_api_knowledge": {
      "enabled": true,
      "sources": [
        {
          "id": 1,
          "api_connection_id": 20
        }
      ]
    }
  }
}
```

### Conversation closure

```json
{
  "components": {
    "conversation_closure": {
      "enabled": true,
      "action_type": "resolve",
      "resolve": {
        "text_message": "Conversation closed"
      },
      "agent_assignment": {
        "is_auto_assign": true,
        "division": {
          "channel_division_id": "division-1"
        },
        "agent": {
          "channel_agent_id": "agent-1"
        }
      }
    }
  }
}
```

Rules:
- `transport="auto"` routes conversation closure to the v3 JSON path
- multipart remains reserved for file-backed attachment uploads

## Practical Selection Rules

1. If the node already exists and the payload is component-heavy, use `workflow_change_node_component`.
2. If the payload contains `attachments[].upload.file_path`, keep `transport="auto"` or explicitly use `multipart`.
3. If the payload contains `conversation_closure`, keep `transport="auto"` unless you intentionally want to reject non-v3 routing.
4. If the task is only simple child creation, stay on `create_bot_response` and avoid advanced component families until a follow-up update step.
