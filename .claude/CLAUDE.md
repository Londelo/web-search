# Web Search MCP

## Project Overview

A self-hosted MCP server that exposes web search and URL fetch capabilities to Claude Code and other MCP-compatible clients. Powered by SearXNG (multi-engine search) and MarkitDown (content extraction).

## Capabilities

- **`calcifer_web_search(query, limit?)`** — Search the web across Google, Bing, DuckDuckGo, Brave, Wikipedia, Reddit, and more. Returns structured results (URL, title, snippet, source engine).
- **`calcifer_web_fetch(url)`** — Fetch a URL and return clean markdown content. Strips HTML noise, handles PDFs, Word docs, and other file types via Python MarkitDown.
- **`calcifer_web_search_within(url, query, limit?)`** — Search within a specific site using the `site:` operator. Enables breadcrumb-style deep navigation.

## Limitations

- **No image support.** Image search and viewing are not implemented. SearXNG's `images` category is available but not wired up.
- **No Cloudflare bypass.** Cloudflare-protected pages will fail with a clear error message directing users to a future stealth browser feature.
- **No social media scraping.** Platforms like Instagram and Facebook require a stealth browser layer and are deferred.
- **No CAPTCHA solving.** Handled by a future stealth browser fallback.

## Architecture

```
Claude Code <---> MCP Server (TypeScript, stdio) <---> SearXNG (Docker, port 8080)
                        |
                        +---> MarkitDown (Python subprocess)
```

## Setup

1. Start SearXNG: `docker compose up -d`
2. Install MarkitDown: `pip3 install markitdown`
3. Install dependencies: `cd mcp-server && npm install`
4. Configure Claude Code: copy `.claude/mcp.json.example` → `.claude/mcp.json`

## Tool Naming

All tools use the `calcifer_` prefix to avoid collision with Claude Code's built-in search/fetch tools.