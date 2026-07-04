# Web Search MCP — Project Design

> **Date:** 2026-07-02
> **Purpose:** Design document for a standalone GitHub repo implementing a web search MCP server using SearXNG + MarkitDown, to be hosted on the user's GitHub account under the name "web search".
> **Scope:** Two MCP tools (calcifer_web_search, calcifer_web_fetch), self-hosted search layer, breadcrumb-style deeper navigation.

---

## Project Overview

A standalone GitHub repo containing an MCP server that exposes web search and URL fetch capabilities to Claude Code and other MCP-compatible clients. The goal is to provide a self-hosted, local-first web search layer.

**Architecture:** MCP Server → SearXNG (search backend) + MarkitDown (content extraction)

One repo, one MCP server, three tools. Simple.

---

## Tools

### `calcifer_web_search(query, limit?)`

General search. Takes a query, returns structured results (URLs, titles, snippets) aggregated across 14 search engines through SearXNG's JSON API.

### `calcifer_web_fetch(url)`

Given a URL, fetches content and returns clean markdown via MarkitDown. Detects YouTube URLs and returns video transcripts instead of page HTML. Truncated to 50K characters.


---

## User Flow

1. **Search:** `calcifer_web_search("how to run local LLMs")` → gets 5-10 URLs with snippets
2. **Pick & Fetch:** Agent picks a URL → `calcifer_web_fetch(url)` → gets clean markdown
3. **Repeat**

---

## Tool Naming

All tools use the `calcifer_` prefix to avoid collision with Claude Code's built-in WebSearch/WebFetch tools.

---

## Tech Stack

| Component | Technology |
|---|---|
| MCP Server | TypeScript (MCP SDK) |
| Search Backend | SearXNG (Docker) |
| Content Extraction | MarkitDown (Python subprocess) |
| YouTube Transcripts | youtube-transcript-api (Python) |
| Testing | Vitest (unit tests) |
| Config | `.claude/mcp.json` (run via `npx tsx`) |

---

## Search Engines (14)

Google, DuckDuckGo, WikiBooks, WikiMedia Search, Microsoft Learn, Arch Wiki, arXiv, Stack Overflow, PyPI, NVD, GitHub, Hugging Face, YouTube, Reddit.

SearXNG queries all enabled engines simultaneously for every search — no per-query engine selection needed.

---

## Limitations

- **No image support.**
- **Instagram/Facebook scraping** — deferred.
- **CAPTCHA solving** — deferred.
- **Session/account management** — not needed.

## Out of Scope / Later

- **`calcifer_web_search_within`** — site-specific search using the `site:` operator. Useful for deep navigation but not a priority for v1.
- **Image search / viewing** — SearXNG supports an `images` category but it's not wired up.

---

## Decisions

- **No npm publishing.** Local-only project, run via `npx tsx`.
- **Python-only for content extraction.** No JS fallback. MarkitDown is a documented prerequisite.
- **Unit tests only.** Vitest with mocked servers.
- **`DEBUG=1` env var** enables stack traces in error responses alongside user-friendly messages.
- **YouTube:** If no transcript exists, return a friendly error with any available metadata (title, duration, channel). No fallback to HTML extraction.
- **Cloudflare-protected pages** will fail with a clear error message (message TBD).
- **Docker Compose at repo root.** SearXNG container at `127.0.0.1:8080`, 512m memory limit.

---

## Phases

### Phase 1: Foundation

**Goal:** Repo scaffolding, TypeScript config, shared types, config module, dependencies.

**Requirements:**
- Repo directory structure with `mcp-server/`, `docker-compose.yml`, `.claude/`, `README.md`
- TypeScript configured (ESM, strict mode, Node 22+ target)
- Shared types for SearXNG responses, tool results, and server config
- Config module that reads from environment variables with sensible defaults
- Dependencies installed: MCP SDK, Zod, TypeScript, tsx, Vitest, ESLint
- Python dependencies installed: markitdown, youtube-transcript-api

### Phase 2: SearXNG Client

**Goal:** HTTP client for SearXNG, response parsing, deduplication, retry, error handling, tests.

**Requirements:**
- Builds and sends queries to SearXNG with proper params (query, format, languages, safesearch)
- Parses JSON responses into typed results
- Deduplicates results by URL — keeps one entry with all source engines attached
- Retries on transient errors (5xx, timeouts, network blips) with exponential backoff, max 2 retries
- No retry on client errors (4xx)
- Timeout enforcement on requests
- Error handling that produces descriptive messages with status codes
- Unit tests covering: valid results, empty results, server errors, timeouts, deduplication, retry success, retry exhaustion

### Phase 3: MCP Server + calcifer_web_search

**Goal:** Server bootstrap, stdio transport, register calcifer_web_search tool.

**Requirements:**
- MCP server bootstraps with name and version, connects via stdio transport
- `calcifer_web_search` tool registered with Zod input validation
  - `query` (required string)
  - `limit` (optional number, default 10, max 30)
- Returns structured results as JSON text with url, title, snippet, source for each result
- Returns `isError: true` with a user-friendly message on failure
- When `DEBUG=1` is set, includes stack trace alongside the user message
- Tool verified via MCP Inspector — shows in tool list, returns correct format

### Phase 4: MarkitDown + YouTube Transcripts

**Goal:** Content extraction via Python subprocess. YouTube transcript support.

**Requirements:**
- Fetches a URL to a temp file, runs MarkitDown on it, captures the markdown output, cleans up
- 60-second subprocess timeout
- YouTube URL detection via common pattern matching (youtube.com/watch, youtu.be/, youtube.com/shorts)
- YouTube transcript fetching via the youtube-transcript-api Python library
- Transcript returned as markdown with timestamps when available
- When no transcript exists, returns a friendly error with available metadata (title, duration, channel)
- Non-YouTube URLs use the standard MarkitDown extraction path
- Unit tests covering: HTML-to-markdown conversion, YouTube transcript success, YouTube no-transcript error, non-YouTube extraction

### Phase 5: calcifer_web_fetch

**Goal:** Wire up calcifer_web_fetch tool with URL validation, content extraction, truncation.

**Requirements:**
- Tool registered with Zod input validation — accepts a required URL string
- URL scheme validation — rejects file://, javascript://, data:// schemes
- Routes YouTube URLs to transcript extraction, everything else to MarkitDown
- Truncates output to 50,000 characters with a clear suffix
- Returns clean markdown as the response
- Handles errors gracefully: HTTP errors, DNS failures, timeouts, subprocess failures — all return `isError: true` with descriptive messages
- When `DEBUG=1` is set, includes stack trace alongside the user message
- Unit tests covering: valid URL, YouTube transcript, YouTube no-transcript error, invalid URL, truncation, HTTP 404

### Phase 6: Docker & Config

**Goal:** SearXNG container setup, settings file, Claude Code MCP config example.

**Requirements:**
- Docker Compose file at repo root with a SearXNG service bound to localhost only
- 512m memory limit, unless-stopped restart policy
- Settings file with all 14 engines enabled, JSON output format, security key configured
- `.claude/mcp.json.example` showing how to point Claude Code at the server
- README documents setup steps (clone, Docker, copy config, restart Claude Code)

### Phase 7: Logging

**Goal:** Local JSONL log file tracking tool usage.

**Requirements:**
- MCP server appends to `.web-search/logs.jsonl` on each tool call
- Each entry includes: timestamp, tool name, input params, result size (bytes), duration (ms), error status
- CLI command `npm run logs` reads the file and prints a readable summary
- No logging in Docker containers, no network calls, no external storage

### Phase 8: Polish

**Goal:** Documentation, linting, final verification.

**Requirements:**
- README with project description, tools reference, engines list, limitations, dev commands
- ESLint configured for TypeScript with strict rules
- All tests pass
- TypeScript compiles without errors
- End-to-end verification: all three tools appear in Claude Code's tool list and are callable

---

## Open Questions

- **Rate limiting** — SearXNG built-in? MCP layer? both?
- **Health check** — SearXNG startup check? graceful failure?