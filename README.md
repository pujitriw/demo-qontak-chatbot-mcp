# demo-mcp-chatbot

Open-source skill pack for operating a Qontak Chatbot MCP server safely through normalized, workflow-first tool usage.

## Why this repo exists

This repository documents and packages practical operating guidance for MCP-based chatbot workflows, with a focus on:

- path and conversation tree operations
- safe branching and recovery flows
- Hub lookups and channel normalization
- consistent tool selection for create, update, publish, and discard flows

Qontak is referenced intentionally because the skill targets Qontak Chatbot MCP workflows.

## Repository layout

- `.agents/skills/qontak-chatbot-mcp/SKILL.md`: primary routing and decision rules
- `.agents/skills/qontak-chatbot-mcp/references/`: detailed setup, payloads, workflows, and troubleshooting guides
- `.claude/skills/qontak-chatbot-mcp`: relative symlink to the same skill for Claude-compatible clients that scan `.claude/skills`

## Quick start

The MCP server is hosted at `https://qontak-chatbot-mcp.triwibowo.com`.

1. Obtain an access token from https://chat.qontak.com/settings/token/omnichannel
2. Point your MCP client at the SSE endpoint with your token in the `Authorization` header:

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

A Streamable HTTP endpoint is also available at `https://qontak-chatbot-mcp.triwibowo.com/mcp` for clients that support it.

3. Review `.agents/skills/qontak-chatbot-mcp/SKILL.md` and load the reference files as project knowledge.

Claude-compatible clients can also discover the same skill via `.claude/skills/qontak-chatbot-mcp`.

For a client-agnostic setup guide covering VS Code, Cursor, Claude Desktop, ChatGPT, and other MCP clients, see `GENERIC_CLIENT_SETUP.md`.

For Claude Desktop step-by-step instructions, see `CLAUDE_DESKTOP_SETUP.md`.

## Security and responsible use

- Never commit real tokens, refresh tokens, or private API base URLs.
- Use placeholder values in docs and examples.
- Report security issues privately as described in `SECURITY.md`.

## Contributing

Contributions are welcome. Read `CONTRIBUTING.md` before opening issues or pull requests.

## License

This project is licensed under the MIT License. See `LICENSE`.
