# @pipeworx/ip-australia

Australian patent, trade mark and design search, plus patent legal status, from IP Australia (the Australian Government's IP office).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1881+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `au_search_trade_marks(query, limit?)` - trade mark search.
- `au_trade_mark(number)` - full public record of one Australian trade mark.
- `au_search_designs(query, limit?)` - registered design search.
- `au_design(number)` - full public record of one Australian design.

## Auth

Platform key with BYO override. The platform client is a free IP Australia OAuth2 client held by Pipeworx; the gateway supplies it as `_apiKey` ("clientId:clientSecret"). A caller may pass their own pair in the same form.

IP Australia is a government register: zero results come back with an explicit note, and an upstream error or an unparsable body is raised, never returned as an empty list.

## Data sources

- IP Australia, Australian patent, trade mark and design registers, and the patent legal-status service. Search is live per request; nothing is copied. No fee applies to read access (IP Australia states there is no fee to connect to its APIs).

The IP Australia API portal publishes request shapes for the trade mark and design searches; the patent search and legal-status request/response shapes are not publicly documented, so those tools pass the register's JSON through unmodified.


Patent search, patent record and patent legal-status tools are built but not listed: IP Australia refuses them to our client until that client is subscribed to those two APIs.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ip-australia": {
      "url": "https://gateway.pipeworx.io/ip-australia/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ip-australia/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1881+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/au_search_trade_marks \
  -H 'Content-Type: application/json' \
  -d '{"query":"TEST","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/au_search_trade_marks`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ip-australia": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ip-australia"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ip-australia
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ip Australia data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
