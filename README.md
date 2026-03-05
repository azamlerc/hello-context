# hello-context

Context repository for the **hello.andrewzc.net** travel agent — an AI-powered chat interface that answers questions about Andrew Zamler-Carhart's travels, geographic interests, and personal website database.

## What this is

This repo contains the context files that define the agent's knowledge and behaviour. It is intentionally public: nothing here is sensitive, and the agent is designed to be embedded on andrewzc.net for anyone to use.

The agent answers questions like:
- Which metro systems in Europe has Andrew completed?
- What confluence points has he visited in France?
- What's near the 48°N 2°E confluence point?
- Which countries is he close to completing?
- What tram systems has he ridden but not finished?

It does this by combining a system prompt (Andrew's voice and travel philosophy) with live calls to the andrewzc REST API at `https://api.andrewzc.net`.

## Files

| File | Purpose |
|---|---|
| `INDEX.md` | Entry point for the agent — file manifest and loading instructions |
| `system-prompt.md` | The agent's system prompt: role, voice, behaviour, constraints |
| `website-intro.md` | Andrew's personal statement about his site and travel philosophy |
| `api-context.md` | API reference: endpoints, field semantics, query patterns |

## How the agent works

The agent is a Claude instance with:
1. A system prompt that gives it Andrew's voice and instructs it how to use the available tools
2. Two context files loaded at session start (`website-intro.md` and `api-context.md`)
3. HTTP tools that call `api.andrewzc.net` to fetch live data from the database

It does not have write access to the database. It answers read-only questions.

## Deployment

The agent is intended to be embedded as a chat widget on `andrewzc.net/hello.html`. API calls to Anthropic and to `api.andrewzc.net` are proxied through a server-side endpoint so no keys are exposed in the browser.

See the `andrewzc-api` repo for the REST API source, and the server-side chat endpoint for the orchestration layer.

## Related repos

| Repo | What it is |
|---|---|
| `andrewzc-api` | REST API serving the website database |
| `andrewzc-mcp-server` | MCP server wrapping semantic search (for Claude Desktop use) |
| `andrewzc-context` | Private context repo for direct Claude sessions (not public) |
# hello-context
