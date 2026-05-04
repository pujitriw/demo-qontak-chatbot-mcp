# Setup

## Purpose

This reference explains the MCP server configuration expected by this skill.

Assume the server is already installed and connected in the host client. If the tools are unavailable, ask the user to connect the Qontak Chatbot MCP server and provide the required environment variables. Do not rely on repository-local files or workspace-specific setup steps.

## Deployment

The MCP server is hosted at:

```
https://qontak-chatbot-mcp.triwibowo.com
```

The transport documented by this skill pack is **SSE**. Clients connect to:

```
https://qontak-chatbot-mcp.triwibowo.com/sse
```

## Authentication

In SSE mode the server acts as a proxy. Credentials are supplied **per-request via HTTP headers** — the server forwards them verbatim to the Chatbot and Hub APIs, which perform the actual validation.

| Header | Description |
|---|---|
| `Authorization` | Access token, e.g. `Bearer <your-access-token>` |
| `X-Refresh-Token` | (recommended) Refresh token for automatic token renewal on 401 |

Tokens can be obtained from: https://chat.qontak.com/settings/token/omnichannel

## Environment Variables (server-side)

These are set on the server itself, not by the client. They do not include credentials in SSE mode.

### Required

- `CHATBOT_API_BASE_URL`

### Recommended

- `HUB_API_BASE_URL` — required when Hub tools are used

### Optional

- `CHATBOT_API_TIMEOUT_SECONDS`
- `HUB_API_TIMEOUT_SECONDS`
- `HUB_API_CLIENT_CONFIG_USE_STAGING`
- `CONTENT_TYPE_CACHE_TTL_SECONDS`
- `LOG_LEVEL`
- `MCP_TRANSPORT` — `sse` (default for this deployment)
- `MCP_HOST` — bind address (default `0.0.0.0`)
- `MCP_PORT` — listen port (default `8000`)

Example server `.env`:

```dotenv
CHATBOT_API_BASE_URL=https://api-chatbot.qontak.com/api
CHATBOT_API_TIMEOUT_SECONDS=30
HUB_API_BASE_URL=https://chat-service.qontak.com/api
HUB_API_TIMEOUT_SECONDS=30
HUB_API_CLIENT_CONFIG_USE_STAGING=false
CONTENT_TYPE_CACHE_TTL_SECONDS=300
LOG_LEVEL=INFO
MCP_TRANSPORT=sse
MCP_HOST=0.0.0.0
MCP_PORT=8000
```

## MCP Client Configuration

### SSE (remote hosted server)

Point the client at the hosted SSE endpoint and supply credentials via headers:

```json
{
  "mcpServers": {
    "qontak-chatbot": {
      "transport": "sse",
      "url": "https://qontak-chatbot-mcp.triwibowo.com/sse",
      "headers": {
        "Authorization": "Bearer replace-with-your-access-token",
        "X-Refresh-Token": "replace-with-your-refresh-token"
      }
    }
  }
}
```

## First Connectivity Check

After the server is connected in the client, start with a safe read call:
- `list_content_types(enabled=true)` if you only need connectivity confirmation
- `get_hub_client_config()` if you need to confirm Hub connectivity first
- `get_path_detail(path_id)` if you already know a valid path id
- `get_path_tree(path_id, preferred_tree_version="v2")` if you need graph visibility immediately

## Runtime Behavior That Affects Tool Usage

### Auth

- In SSE mode, credentials come from the `Authorization` and `X-Refresh-Token` request headers and are scoped to the individual connection
- If the upstream API returns `401` and a refresh token is present, the server calls `/oauth/token`, updates the in-memory token for that connection only, and retries the original request once

### Tree reads

- tree reads prefer `v2`
- fallback order is `v2`, then `v1`, then `v3`
- the actual version used is returned in `meta.tree_version_used`

### Hub routing

- Hub tools use `HUB_API_BASE_URL`
- `get_hub_client_config` may use a staging override when `HUB_API_CLIENT_CONFIG_USE_STAGING=true`
- keep that override scoped to client-config lookup only unless the user explicitly wants different behavior

### Content type resolution

- content types are resolved dynamically from `/v1/content_types`
- the server uses a short-lived cache
- unresolved content type code or name forces one refresh before returning a validation error
