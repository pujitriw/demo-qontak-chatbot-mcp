# Generic MCP Client Setup Guide

This guide shows how to connect a generic MCP client to the Qontak Chatbot MCP server and load the `demo-mcp-chatbot` skill pack.

It is written for clients such as:

- VS Code
- Cursor
- Claude Desktop
- ChatGPT clients that support custom MCP servers
- Antigravity and other MCP-capable desktop clients

The exact UI labels vary by client, but the connection details are the same.

## Transport

This guide only covers the hosted SSE connection, which is the recommended setup for this demo.

Hosted SSE endpoint:

- SSE: `https://qontak-chatbot-mcp.triwibowo.com/sse`

## What You Need

Before configuring any client, collect these values:

1. An access token from `https://chat.qontak.com/settings/token/omnichannel`
2. An optional refresh token from the same page
3. This repository, if you want to load the skill pack into your client as project knowledge or attachments

Security notes:

- Do not commit real tokens into `mcp.json`, `settings.json`, or any repo config.
- Keep the `Bearer ` prefix in the `Authorization` header.
- `X-Refresh-Token` is optional but recommended.

## Generic SSE Setup

If your client supports SSE MCP servers, the connection usually looks like this:

```json
{
  "servers": {
    "qontak-chatbot": {
      "type": "sse",
      "url": "https://qontak-chatbot-mcp.triwibowo.com/sse",
      "headers": {
        "Authorization": "Bearer replace-with-your-access-token",
        "X-Refresh-Token": "replace-with-your-refresh-token"
      }
    }
  }
}
```

Some clients use `mcpServers` or nest the transport under `transport`. If so, keep the same values and adapt only the wrapper keys.

## Client Examples

## VS Code

VS Code commonly stores MCP servers in a user `mcp.json` file.

Example shape:

```json
{
  "servers": {
    "qontak-chatbot": {
      "type": "sse",
      "url": "https://qontak-chatbot-mcp.triwibowo.com/sse",
      "headers": {
        "Authorization": "Bearer replace-with-your-access-token",
        "X-Refresh-Token": "replace-with-your-refresh-token"
      }
    }
  },
  "inputs": []
}
```

This matches the structure already used in VS Code MCP configs such as `mcp.json` with a top-level `servers` object.

## Cursor

Cursor MCP configuration may use a different wrapper, but the important fields stay the same:

- server name: `qontak-chatbot`
- transport type: `sse`
- URL details
- `Authorization` header
- optional `X-Refresh-Token`

Use the hosted SSE shape.

## Claude Desktop

Claude Desktop often uses `mcpServers` and may nest the transport details.

Hosted example:

```json
{
  "mcpServers": {
    "qontak-chatbot": {
      "transport": {
        "type": "sse",
        "url": "https://qontak-chatbot-mcp.triwibowo.com/sse",
        "headers": {
          "Authorization": "Bearer replace-with-your-access-token",
          "X-Refresh-Token": "replace-with-your-refresh-token"
        }
      }
    }
  }
}
```

For a Claude-only walkthrough, see `CLAUDE_DESKTOP_SETUP.md`.

## ChatGPT

If your ChatGPT client or plan supports custom MCP servers, use the hosted server details from this guide and map them into the client's MCP server form.

Use these values:

- name: `qontak-chatbot`
- SSE URL: `https://qontak-chatbot-mcp.triwibowo.com/sse`
- `Authorization: Bearer ...`
- optional `X-Refresh-Token`

## Antigravity

For Antigravity or similar MCP desktop clients, create a new MCP server using the same hosted settings:

- name: `qontak-chatbot`
- type: `sse`
- url: `https://qontak-chatbot-mcp.triwibowo.com/sse`
- header `Authorization`: `Bearer replace-with-your-access-token`
- header `X-Refresh-Token`: `replace-with-your-refresh-token`

## Other MCP Clients

For any other MCP-capable client:

1. Create a new custom MCP server
2. Choose `sse`
3. Reuse the exact same token headers
4. Save and reload the client

The wrapper keys differ by client, but the payload values do not.

## Verify Connectivity

After adding the server, restart or reload your client if needed.

Then run a harmless read-only call such as:

```text
Call list_content_types with enabled=true and show me the result.
```

Success means:

- the `qontak-chatbot` server loads
- the client can see the MCP tools
- the request completes without an auth or startup error

## Load the Demo Skill Pack

This repository contains a skill pack that helps an agent use the server correctly.

Recommended files to load as project knowledge, instructions, or attachments:

- `.agents/skills/qontak-chatbot-mcp/SKILL.md`
- `.agents/skills/qontak-chatbot-mcp/references/setup.md`
- `.agents/skills/qontak-chatbot-mcp/references/tool-catalog.md`
- `.agents/skills/qontak-chatbot-mcp/references/workflows.md`
- `.agents/skills/qontak-chatbot-mcp/references/payloads.md`
- `.agents/skills/qontak-chatbot-mcp/references/troubleshooting.md`
- `.agents/skills/qontak-chatbot-mcp/references/hub-tools.md`
- `.agents/skills/qontak-chatbot-mcp/references/component-updates.md`

Suggested instruction text:

```text
When working with Qontak Chatbot paths, conversations, branches, or Hub operations, use the connected qontak-chatbot MCP server first and follow the uploaded qontak-chatbot skill files as the operating guide.
```

## Troubleshooting

If the server does not appear:

- check that your client supports the selected transport type
- verify the token still works in Qontak
- keep the `Bearer ` prefix in the header value
- restart the client after editing the config

If requests fail with auth errors:

- refresh the access token
- add or update `X-Refresh-Token`
- confirm the client is actually sending custom headers