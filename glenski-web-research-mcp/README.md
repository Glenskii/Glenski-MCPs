# glenski-web-research-mcp

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://python.org)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-orange.svg)](https://creativecommons.org/licenses/by/4.0/)
[![No API Key](https://img.shields.io/badge/API%20key-none%20required-brightgreen.svg)](#no-api-keys)
[![MCP Compatible](https://img.shields.io/badge/MCP-Claude%20Code%20%7C%20Desktop-blueviolet.svg)](https://modelcontextprotocol.io)

**Live web research for Claude. No API key. No vendor lock-in. No rate bills.**

Three tools — `web_search`, `fetch_page`, `multi_search` — turn Claude into a live research engine that searches, reads, and cross-references before answering. Powered by DuckDuckGo, httpx, and BeautifulSoup. Works with Claude Code, Claude Desktop, and any MCP-compatible host.

Part of [Glenski-MCPs](https://github.com/Glenskii/Glenski-MCPs).

---

## Tools

### `web_search`

Full-web DuckDuckGo search. Returns titles, URLs, and snippets with optional region and recency filtering. Retries automatically on rate limit errors using exponential backoff with jitter.

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `query` | str | required | Search query string |
| `max_results` | int | 5 | 1–10 results |
| `region` | str | `wt-wt` | Worldwide. Use `us-en`, `ca-en`, `gb-en`, etc. |
| `time_filter` | str | None | `d` day · `w` week · `m` month · `y` year |

**Response:**
```json
{
  "query": "Cloudflare Workers limits 2025",
  "timestamp": "2025-07-12T14:22:01Z",
  "result_count": 5,
  "results": [
    {
      "title": "Workers limits - Cloudflare Docs",
      "url": "https://developers.cloudflare.com/workers/platform/limits/",
      "snippet": "Workers scripts have a 1 MB size limit on the Free plan...",
      "published": ""
    }
  ]
}
```

---

### `fetch_page`

Fetches any URL and returns clean, readable body text. Strips navigation, ads, scripts, sidebars, footers, and structural noise before returning content. Prefers `<article>` and `<main>` elements when present.

**JS-rendered page detection:** If a page returns HTTP 200 but very few words (under 80), the real content is almost certainly rendered client-side via JavaScript. The tool returns `js_rendered_hint: true` with a note directing Claude to route that URL to Playwright MCP for full rendered content.

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `url` | str | required | Full URL including `https://` |
| `max_chars` | int | 8000 | Body text character limit |

**Response (static page):**
```json
{
  "url": "https://developers.cloudflare.com/workers/platform/limits/",
  "title": "Workers limits · Cloudflare Docs",
  "text": "CPU time\nFree plan: 10ms. Paid plan: 30s...",
  "word_count": 1842,
  "status_code": 200,
  "js_rendered_hint": false,
  "timestamp": "2025-07-12T14:22:04Z"
}
```

**Response (JS-rendered page):**
```json
{
  "url": "https://some-react-app.com/pricing",
  "title": "Pricing",
  "text": "",
  "word_count": 12,
  "status_code": 200,
  "js_rendered_hint": true,
  "timestamp": "2025-07-12T14:22:04Z",
  "note": "Low word count on a successful fetch. This page is likely JS-rendered. Use Playwright MCP to fetch this URL instead."
}
```

---

### `multi_search`

Runs 2–5 independent search queries **simultaneously** via `asyncio` + `ThreadPoolExecutor`. All queries fire at the same time — no sequential delays. Use when a topic benefits from multiple angles: conflicting claims, different phrasings that surface different results, or comparative research.

On 3 queries this is roughly 3x faster than running them one after another. Each query still retries with backoff on DDG rate limits.

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `queries` | list[str] | required | 2–5 query strings |
| `max_results_each` | int | 3 | 1–5 results per query |
| `region` | str | `wt-wt` | Same as `web_search` |
| `time_filter` | str | None | Same as `web_search` |

**Response:**
```json
{
  "timestamp": "2025-07-12T14:22:06Z",
  "query_count": 3,
  "total_results": 9,
  "results_by_query": {
    "Cloudflare Workers performance 2025": { "result_count": 3, "results": [...] },
    "Vercel Edge Functions benchmark 2025": { "result_count": 3, "results": [...] },
    "Workers vs Vercel comparison": { "result_count": 3, "results": [...] }
  }
}
```

---

## Why this exists

Every popular web-research MCP ties Claude to a specific paid API — Perplexity, Brave, Bing, or SerpAPI. This one does not. DuckDuckGo requires no account, no key, and no billing. httpx and BeautifulSoup run locally. Nothing phones home.

**Comparison:**

| | This MCP | Perplexity MCP | Brave MCP | SerpAPI |
|---|---|---|---|---|
| API key required | No | Yes | Yes | Yes |
| Cost | Free | Per-query after free tier | Free tier limited | Per-query |
| Page fetching | Yes | No | No | No |
| Parallel search | Yes | — | — | — |
| JS-page detection | Yes | — | — | — |
| Runs locally | Yes | No | No | No |

If you later want to layer in a paid search API for higher volume or better ranking, the architecture supports it. Add the key as an environment variable and wire a new tool in `server.py`.

---

## Install

### 1. Clone and install dependencies

```bash
git clone https://github.com/Glenskii/Glenski-MCPs
cd Glenski-MCPs/glenski-web-research-mcp
pip install -r requirements.txt
```

**Using a virtual environment (recommended):**
```bash
git clone https://github.com/Glenskii/Glenski-MCPs
cd Glenski-MCPs/glenski-web-research-mcp
python -m venv venv

# Windows:
.\venv\Scripts\activate
# macOS / Linux:
source venv/bin/activate

pip install -r requirements.txt
```

When using a venv, point `mcp.json` at the venv Python, not system Python — see the config examples below.

---

### 2a. Claude Code

Edit `~/.claude/mcp.json` (create it if it does not exist):

**Standard — system Python:**
```json
{
  "mcpServers": {
    "glenski-web-research": {
      "command": "python",
      "args": ["C:/path/to/Glenski-MCPs/glenski-web-research-mcp/server.py"]
    }
  }
}
```

**With virtual environment:**
```json
{
  "mcpServers": {
    "glenski-web-research": {
      "command": "C:/path/to/Glenski-MCPs/glenski-web-research-mcp/venv/Scripts/python.exe",
      "args": ["C:/path/to/Glenski-MCPs/glenski-web-research-mcp/server.py"]
    }
  }
}
```

Replace `C:/path/to` with your actual clone location. Restart Claude Code — the server appears in your connected MCP list.

---

### 2b. Claude Desktop

Edit your config file:

- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`

Add the same `mcpServers` block with the corrected path and restart.

---

### 2c. Cursor / Windsurf

Add the same `mcpServers` block to your editor's MCP config file. Both editors use the same JSON format.

---

### 3. Verify

Ask Claude:

> *"Search the web for the latest news on Model Context Protocol and summarize what you find."*

Claude will call `web_search`, optionally `fetch_page` on the top results, and return a sourced response with URLs and access timestamps.

---

## Usage Patterns

**Current events with recency filter:**
```
What's happened with WordPress security vulnerabilities in the last month?
```
Claude calls `web_search(query, time_filter="m")` — results from the past 30 days only.

**Deep read of a specific page:**
```
Get the full content of this Cloudflare pricing page and summarize the plan limits.
```
Claude calls `fetch_page(url)`, which returns clean content with all nav and ad noise stripped.

**Multi-angle comparative research — parallel:**
```
Compare what different sources say about Cloudflare Workers vs Vercel Edge Functions for latency.
```
Claude calls `multi_search(["Cloudflare Workers latency 2025", "Vercel Edge Functions performance", "Workers vs Vercel benchmark"])`. All three queries fire at the same time.

**JS-rendered page fallback:**
```
Get the pricing tiers from https://some-react-app.com/pricing
```
Claude calls `fetch_page(url)`. If `js_rendered_hint: true` fires, Claude automatically routes to Playwright MCP for the fully rendered content — no manual intervention needed.

**Regional search:**
```
What are Canadian photographers saying about AI copyright law changes?
```
Claude calls `web_search(query, region="ca-en")` — DuckDuckGo returns Canada-region results.

---

## Research Protocol

The MCP's system instructions embed a behavioral protocol for all tool use. Claude applies it automatically when this server is connected:

1. **Search first** — run `web_search` before forming any answer to factual queries
2. **Fetch sources** — use `fetch_page` on the top 2–3 results for full content, not just snippets
3. **Parallelize** — use `multi_search` for topics needing multiple angles (all queries fire simultaneously)
4. **Cite everything** — include URL and access timestamp for every source used
5. **Flag conflicts** — note disagreements between sources explicitly
6. **Rate confidence** — High / Medium / Low based on source consensus and recency
7. **JS fallback** — if `fetch_page` returns `js_rendered_hint: true`, route that URL to Playwright MCP

This protocol is derived from the **Web Research Prompt** by Glen E. Grant — a standalone research methodology field-tested as an AI prompt system before being encoded as an MCP.

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'ddgs'`**
The DuckDuckGo package was renamed in 2025. Run: `pip install ddgs`

**`ModuleNotFoundError: No module named 'mcp'`**
Run: `pip install mcp`

**Search returns empty results**
DuckDuckGo applies rate limits under heavy use. The server retries automatically with exponential backoff. If the problem persists, wait 30–60 seconds and retry. Avoid firing many simultaneous searches.

**`fetch_page` returns almost no text**
The page is JavaScript-rendered. Check for `js_rendered_hint: true` in the response. Route the URL to Playwright MCP instead.

**Server doesn't appear in Claude Code**
Check that the path in `mcp.json` uses forward slashes (`/`) or escaped backslashes (`\\`). Verify Python is in your system PATH or use the absolute path to the Python executable.

**Claude ignores the tools and answers from memory**
The server's instructions guide tool use for factual queries, but Claude may default to training data on simple or conversational questions. Ask explicitly: *"Search the web and tell me..."* to trigger tool use.

---

## Requirements

```
python >= 3.10
mcp >= 1.0.0
ddgs >= 0.1.0
httpx >= 0.27.0
beautifulsoup4 >= 4.12.0
```

> `ddgs` is the current package name for DuckDuckGo search, renamed from `duckduckgo-search` in 2025.

---

## No API Keys

Zero vendor dependencies. The `env: {}` block in your MCP config is intentionally empty — this server needs nothing from the environment.

---

## Changelog

### v2.0
- `multi_search` now runs all queries in parallel via asyncio + ThreadPoolExecutor (was sequential with 0.75s delays between queries)
- Exponential backoff with jitter added to all DuckDuckGo search calls
- `fetch_page` now returns `js_rendered_hint: true` on likely JS-rendered pages, with a note directing Claude to Playwright MCP
- Added Claude Code install instructions alongside Claude Desktop

### v1.0
- Initial release: `web_search`, `fetch_page`, `multi_search` via DuckDuckGo + httpx + BeautifulSoup

---

## Origin and Credits

This MCP is the executable form of the **Web Research Prompt** built by Glen E. Grant.

The core research protocol embedded here — tool priority order, mandatory source fetching, parallel multi-angle cross-referencing, confidence rating, and citation structure — was designed and field-tested by Glen as a standalone AI prompt system before being encoded as an MCP. The `multi_search` tool directly implements his parallel-query cross-referencing methodology.

**Author:** [Glen E. Grant](https://glenegrant.com) · [glen@glenegrant.com](mailto:glen@glenegrant.com) · [github.com/Glenskii](https://github.com/Glenskii)

---

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — share freely, credit appreciated.

---

*Part of [Glenski-MCPs](https://github.com/Glenskii/Glenski-MCPs) — practical MCP tools built for real workflows.*
