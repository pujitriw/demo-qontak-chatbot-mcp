# Contributing

Thanks for helping improve this project.

## Before you start

- Read `.agents/skills/qontak-chatbot-mcp/SKILL.md` first.
- Keep changes aligned with the workflow-first design in the skill.
- Do not add undocumented tools or payload shapes without updating references.

## Development guidelines

- Prefer clear, minimal edits.
- Keep examples sanitized (no real credentials, tokens, or private endpoints).
- Preserve terminology consistency: `path` in tools, `Conversation` in UI wording.
- Update related reference docs when behavior changes.

## Pull request checklist

- Explain what changed and why.
- Note any behavior changes in workflow or payload expectations.
- Confirm docs and examples still use placeholders for secrets.
- Add tests if the repo introduces executable code in the future.

## Reporting issues

- Use issues for bugs, docs fixes, and enhancement proposals.
- For security reports, follow `SECURITY.md` instead of public issues.
