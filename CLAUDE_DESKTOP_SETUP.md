# Claude Desktop Setup Guide

This guide shows how to connect Claude Desktop to the hosted Qontak Chatbot MCP server and load the `demo-mcp-chatbot` skill pack.

## What You Will Set Up

By the end of this guide you will have:

1. configured Claude Desktop to connect to the Qontak Chatbot MCP server
2. verified that Claude can see the MCP tools
3. loaded the demo skill files into Claude Desktop as project knowledge

## Demo skill repo

- Use this repository checkout.

---

## Part 1: Connect via Hosted SSE

No local server install is needed. You only need a valid access token.

### Step 1: Get your access token

Obtain an access token and, optionally, a refresh token from:

```
https://chat.qontak.com/settings/token/omnichannel
```

Keep both values ready — you will paste them into the Claude Desktop config.

### Step 2: Configure Claude Desktop

On macOS, Claude Desktop reads MCP servers from:

```text
~/Library/Application Support/Claude/claude_desktop_config.json
```

Open this file from Claude Desktop:

1. Open Claude Desktop.
2. In the macOS menu bar, click `Claude`.
3. Open `Settings`.
4. Go to `Developer`.
5. Click `Edit Config`.

Replace or merge in the following configuration. Substitute your real token values.

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

Important notes:

- The `Authorization` value must be the full header value, including the `Bearer ` prefix.
- `X-Refresh-Token` is optional but recommended. When present, the server will automatically refresh your token on a `401` response and retry the request.
- Credentials are sent to the server with every request and forwarded verbatim to the Chatbot and Hub APIs, which perform the actual validation. They are not stored server-side.

### Step 3: Restart Claude Desktop

After saving the JSON config:

1. Fully quit Claude Desktop.
2. Open Claude Desktop again.
3. Start a new chat.

If the server loads successfully, Claude Desktop will show the MCP tools as available in the chat UI.

### Step 4: Verify connectivity

In a fresh Claude chat, run a harmless read-only call first:

```text
Call list_content_types with enabled=true and show me the result.
```

What success looks like:

- Claude sees the `qontak-chatbot` MCP server
- Claude returns a tool response without a startup or auth error

---

## Part 2: Load the Demo Skill Pack

This applies to the hosted SSE setup.

Claude Desktop can connect to MCP servers directly, but it does not have a native import flow for skill folders. Load the skill files as Project knowledge.

### Recommended: use a Claude Project

1. Open Claude Desktop.
2. Create a new Project, for example `Qontak Chatbot MCP`.
3. Open the Project knowledge area.
4. Upload the skill files from the `demo-mcp-chatbot` repository:

- `.agents/skills/qontak-chatbot-mcp/SKILL.md`
- `.agents/skills/qontak-chatbot-mcp/references/setup.md`
- `.agents/skills/qontak-chatbot-mcp/references/tool-catalog.md`
- `.agents/skills/qontak-chatbot-mcp/references/workflows.md`
- `.agents/skills/qontak-chatbot-mcp/references/payloads.md`
- `.agents/skills/qontak-chatbot-mcp/references/troubleshooting.md`
- `.agents/skills/qontak-chatbot-mcp/references/hub-tools.md`
- `.agents/skills/qontak-chatbot-mcp/references/component-updates.md`

### Suggested project instruction

```text
When working with Qontak Chatbot paths, conversations, branches, or Hub operations, use the connected qontak-chatbot MCP server first and follow the uploaded qontak-chatbot skill files as the operating guide.
```

### Alternative: attach files in a chat

Attach the same markdown files directly to a conversation. Less reusable than Project knowledge — you will need to reattach them in future chats.

## Full workflow test

Once the server is connected and the skill files are loaded:

```text
Use the qontak-chatbot MCP tools. First verify connectivity with list_content_types(enabled=true). Then explain which tool family I should use if I want to create a new Conversation with the first reply.
```

---

## Troubleshooting

### Claude Desktop does not show the server

- confirm the JSON is valid
- confirm the `url` value is `https://qontak-chatbot-mcp.triwibowo.com/sse`
- confirm the `Authorization` header value includes the `Bearer ` prefix
- fully restart Claude Desktop after any config change

### Tool calls fail with 401

- your access token has expired — generate a new one at https://chat.qontak.com/settings/token/omnichannel
- update the `Authorization` header value in the Claude Desktop config and restart

### The skill files are uploaded, but Claude ignores them

- confirm the files were uploaded into the same Project you are chatting in
- make your instruction more explicit:

```text
Always use the uploaded qontak-chatbot skill files as the decision guide for tool selection and workflow sequencing.
```

### I want Claude Desktop to read files from the repositories directly

That is separate from this MCP server.

If you want Claude Desktop to browse local folders directly, add a filesystem-style MCP server as well. The `qontak-chatbot` server only provides Qontak Chatbot and Hub tools. It does not expose general file browsing tools.

## Recommended Daily Workflow

After setup, a practical daily workflow looks like this:

1. open Claude Desktop
2. open the Project that contains the demo skill files
3. start a chat in that Project
4. ask Claude to use the `qontak-chatbot` MCP tools
5. begin with a read-only verification call before any mutation

Example prompt:

```text
Use the qontak-chatbot MCP server and the uploaded qontak-chatbot skill pack. Start by checking connectivity, then help me inspect a Conversation safely before making any edits.
```

## Summary

- Claude Desktop connects directly to the hosted `qontak-chatbot` MCP server over SSE
- `demo-mcp-chatbot` is the skill and reference pack you load into Claude as project knowledge or chat attachments
- the MCP server provides the tools, and this repository provides the operating guidance
