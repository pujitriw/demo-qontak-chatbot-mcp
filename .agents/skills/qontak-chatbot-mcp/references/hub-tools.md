# Hub Tools

## Purpose

This reference documents the Hub MCP tools, their default payloads, their recommended ordering, and the quirks that matter for tool selection.

Use it when the task needs Hub lookups or uploads before a chatbot edit flow, especially for auth bootstrap, integration selection, assignment lists, tag lists, Qontak CRM lookup, or attachment uploads.

## Shared Rules

- Hub tools use the same MCP response envelope as the chatbot tools
- Hub tools target `HUB_API_BASE_URL`, not `CHATBOT_API_BASE_URL`
- auth stays shared with `CHATBOT_API_TOKEN` and `CHATBOT_API_REFRESH_TOKEN`
- Hub responses should report `meta.api_version_used = "hub-core-v1"`
- `tree_version_used` does not apply to Hub tools

## Tool Notes

### `get_hub_user_profile`

- endpoint: `GET /core/v1/users/me`
- common role: auth bootstrap
- defaults: none
- result keys: `profile`

### `list_hub_users`

- endpoint: `GET /core/v1/users`
- default payload: `{ offset: 1, limit: 10 }`
- common role: assignment and agent-pick flows
- result keys: `users`, `pagination`

### `get_hub_organization_me`

- endpoint: `GET /core/v1/organizations/me`
- common role: organization bootstrap and settings flows
- quirk: some clients attach tag-style params to this call; treat that as a caller quirk, not as a required semantic default
- result keys: `organization`

### `list_hub_integrations`

- endpoint: `GET /core/v1/integrations`
- default payload: `{ offset: 1, limit: 100 }`
- common role: integration selection before path or component editing
- quirk: if a caller produces a nested or malformed pagination shape, normalize back to the flat payload above
- result keys: `integrations`, `pagination`

### `list_hub_divisions`

- endpoint: `GET /core/v1/divisions`
- default payload: `{ offset: 1, limit: 100 }`
- common role: assignment, routing, and division pickers
- lookup nuance: if a saved id is missing from the first page, retry with `query=<id>` and a narrow limit
- result keys: `divisions`, `pagination`

### `list_hub_tags`

- endpoint: `GET /core/v1/tags`
- default payload: `{ offset: 1, limit: 100, query: "" }`
- common role: tag pickers before node updates
- result keys: `tags`, `pagination`

### `upload_hub_message_file`

- endpoint: `POST /core/v1/file_uploader/message`
- common role: attachment preparation before chatbot node updates
- transport rule: keep the MCP contract on explicit file-content inputs and let the server translate them to multipart upstream
- result keys: `upload`

### `get_hub_qontak_integration_uniq`

- endpoint: `GET /core/v1/qontak/integration/uniq`
- common role: Qontak CRM preparation before CRM-related node edits
- defaults: none confirmed; pass through caller payload only
- result keys: `integration`

### `get_hub_billing_info`

- endpoint: `GET /core/v1/billings/info`
- common role: auth bootstrap and settings flows
- defaults: none confirmed; pass through caller payload only
- result keys: `billing`

### `get_hub_client_config`

- endpoint: `GET /core/v1/client_configs/config`
- common role: first Hub call in auth bootstrap and any flow that depends on client-config flags
- staging nuance: a client-config-only staging override may be enabled through MCP server configuration
- seamless-auth nuance: if the response enables `seamless_auth`, treat that response as a prerequisite for later Hub calls that depend on auth behavior
- result keys: `config`

## Recommended Lookup Order

### Auth bootstrap

1. `get_hub_client_config`
2. `get_hub_user_profile`
3. `get_hub_organization_me`
4. `get_hub_billing_info`

Use this order when the task needs Hub bootstrap state before profile-dependent actions.

### Conversation setup

1. `list_hub_integrations`
2. choose the integration in the editor or drawer flow
3. continue with chatbot path or conversation tools

Use this order when the task needs channel selection before path create or path update.

### Assignment and tagging

1. `list_hub_tags`
2. `list_hub_divisions`
3. `list_hub_users`

Use this order when a node edit depends on tags, divisions, and users.

### Attachment flow

1. `upload_hub_message_file`
2. use the returned upload metadata in the chatbot bot-response update flow

### Qontak CRM flow

1. `get_hub_qontak_integration_uniq`
2. continue with the CRM or component edit that depends on the uniq state