---
date: 2026-05-19
ticket: none
repo: ybor-cortex
status: approved
---

# Plan: Cortex MCP Server — Platform Knowledge via Claude Code

## Context

Cortex has a RAG-powered knowledge base with ~1000+ platform knowledge articles indexed from internal repos. Currently, this knowledge is only accessible through the web console UI (`/discover` page). The user wants to access Cortex knowledge directly from Claude Code (and any other MCP client) without opening a browser.

**Two-part approach**:
1. **Part 1**: Create an MCP server wrapping Cortex's existing search — opens up knowledge anywhere
2. **Part 2**: Configure Claude Code to use the MCP server — universal console access

## Architecture

The MCP server is a **lightweight standalone process** that reuses existing cortex-api code:
- Connects directly to the same PostgreSQL + pgvector database
- Uses the same OpenAI embedding model (`text-embedding-3-large`)
- Reuses `KnowledgeSearch` class for vector similarity search
- Runs as a stdio subprocess launched by Claude Code

## Tools Exposed

1. **`search_knowledge`** — HyDE-enhanced vector similarity search over the platform knowledge base
2. **`list_knowledge_categories`** — List available knowledge categories with article counts

## Key Files

| File | Action | Description |
|------|--------|-------------|
| `cortex-api/cortex_mcp_server/server.py` | New | FastMCP server with tool definitions |
| `cortex-api/cortex_mcp_server/db.py` | New | Async DB session factory (standalone) |
| `cortex-api/pyproject.toml` | Edit | Add `mcp[cli]` dependency |
| `.claude/settings.local.json` | Edit | Add MCP server config |

## Implementation Order

1. Add `mcp[cli]` dependency to `pyproject.toml`
2. Create `cortex_mcp_server/` package (db.py, server.py, __init__.py)
3. Test locally with `uv run python -m cortex_mcp_server.server`
4. Configure Claude Code (`settings.local.json`)
5. Update `docs/cortex-status.md`
6. Commit everything together
