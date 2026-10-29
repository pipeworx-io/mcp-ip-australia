# @pipeworx/ip-australia

Australian patent, trade mark and design search, plus patent legal status, from IP Australia (the Australian Government's IP office).

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `au_search_trade_marks(query, limit?)` - trade mark search.
- `au_trade_mark(number)` - full public record of one Australian trade mark.
- `au_search_designs(query, limit?)` - registered design search.
- `au_design(number)` - full public record of one Australian design.
- `au_patent(number)` - full public record of one Australian patent, including its
  `ipRightStatusEvent` history (FILED / OPI / ACCEPTED / SEALED with dates).

Built but not served (see "What is not shipped, and why" below):
`au_search_patents`, `au_legal_status`.

## Auth

Platform key with BYO override. The platform client is a free IP Australia OAuth2 client held by Pipeworx; the gateway supplies it as `_apiKey` ("clientId:clientSecret"). A caller may pass their own pair in the same form.

Each API product is subscribed separately on the IP Australia developer portal, and a client that is not subscribed to one gets HTTP 403 `Invalid Client` on it with a perfectly valid token. So an entitlement failure looks nothing like an auth failure, and the refusal names the environment and the status rather than guessing.

## What is not shipped, and why

Measured through the deployed gateway on 2026-10-27T01:31Z with a valid production token. Two tools are held in `deferredTools`, and the two failures are different in kind:

| Tool | Live result | What it means |
|---|---|---|
| `au_legal_status` | HTTP 403 `Invalid Client` | **Entitlement.** The production client is not subscribed to the Legal Status API product. Nothing in this repo fixes it. |
| `au_search_patents` | HTTP 400 `Bad request` | **Our request body.** Patent Search *is* subscribed -- `au_patent` returns a full record through the same product and the same token. The patent quick-search body shape is undocumented and the shapes differ per API (trade marks need `filters: {quickSearchType: ['WORD']}`; designs answer HTTP 500 if any `filters` object is present), so this needs body shapes tried against the live endpoint. |

Before anyone buys the Legal Status product: `au_patent` already carries `ipRightStatusEvent` -- the filing, open-to-public-inspection, acceptance and sealing events with their dates -- so most of the legal-status question is already answered by the patent record.

IP Australia is a government register: zero results come back with an explicit note, and an upstream error or an unparsable body is raised, never returned as an empty list.

## Personal data: no individual addresses

The Australian registers are public, and many trade marks and designs are held by natural persons rather than companies — for those the owner record carries a **home address**: street, suburb, state and postcode.

Bruce ruled `no-individual-addresses` on this pack (2026-10-26), on the same terms he set for `sos-entities` and `exec-changes` under `names-and-roles` (2026-10-08). So:

- **Names are served.** Owner, applicant, designer and inventor names come through as the register publishes them, with the publisher and dataset named in every response's `source`.
- **An address is served only for a party positively identified as an ORGANISATION.** Positive identification means one of: a registered-business identifier on the party (ABN, ACN or ARBN); a legal-entity or firm designator as a whole word in the name (`Pty`, `Ltd`, `Limited`, `Inc`, `LLC`, `GmbH`, `SARL`, `University`, `Council`, `Patent Attorneys`, `Intellectual Property`, and so on); or a dotted/slashed legal abbreviation in the name (`N.V.`, `B.V.`, `S.A.`, `A/S`, `L.L.C.`).
- **Every other party loses its street address.** The name and the COUNTRY survive — a country is a jurisdiction, not a place a person can be found at — and the street, suburb, state/territory and postcode do not.
- **A designer or inventor never gets an address**, designator in the name or not, because the register records that party as a natural person by definition.

### Which way the line fails, and why it is drawn there

The test is "is this positively an organisation", not "does this look like a person". Those are not the same rule and they fail in opposite directions.

A name-shape heuristic was tried first and, measured against the live register while building this, read `Societe Pour La Transformation Des Matieres Plastiques Stamp` — a real company — as a person. The two errors are not symmetric:

- A **false positive** (a company read as possibly-a-person) withholds a business address a caller could have had. It costs one field that the register itself still publishes.
- A **false negative** (a person read as a company) serves a named individual's home address. That is the exact thing the ruling forbids, and once served it cannot be recalled.

So the default is to withhold, and only a positive organisation signal serves. The consequence is deliberate and worth stating plainly: **a company whose name carries no designator we recognise keeps its name and loses its address.** That is the price of never getting the other error.

For the same reason no designator shorter than three letters is used. The obvious list also holds `NV`, `BV`, `SA`, `AG`, `AB`, `AS`, `OY`, `KK`, `CO`, `PT` — and each of those is also a word in somebody's transliterated name (`Nguyen Van Co`, `Tran Thi Sa`). Those suffixes are recognised only in their dotted form, which a personal name does not carry.

### It is announced, never silent

A withheld address and a register record that never held one must not look the same to a caller. So the party carries `address_withheld: "<why>"`, and the response carries `addresses_withheld: <n>` plus a `privacy_note`. Nothing is quietly dropped.

### Removal list

Names are withheld on request, as the ruling requires. The list is `src/removals.ts`; it ships with the package, so it holds wherever the pack runs, including a standalone install with no database. A removal may be scoped to one record number or apply to every record this pack serves. Requests go to support@pipeworx.io.

## Data sources

- IP Australia, Australian patent, trade mark and design registers, and the patent legal-status service. Search is live per request; nothing is copied. No fee applies to read access (IP Australia states there is no fee to connect to its APIs).

The IP Australia API portal publishes request shapes for the trade mark and design searches; the patent search and legal-status request/response shapes are not publicly documented, so those tools pass the register's JSON through unmodified — after the personal-data filter, which walks the whole payload rather than a list of known field paths precisely because those shapes are undocumented.

That caution earned itself on the first live patent record: the patent register names its parties with `legalEntityName` rather than `name` (`applicant`, `inventors` and `representatives` are all name-only objects), which a field-path filter written against the trade mark and design shapes would not have recognised as parties at all. None of them carried an address on the records read, so nothing leaked — but the filter now reads every name key the registers use, rather than the two we happened to see first.

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

Both URLs reach the same gateway and the same 1856+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/au_search_trade_marks \
  -H 'Content-Type: application/json' \
  -d '{"query":"TEST","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/au_search_trade_marks`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-ip-australia` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/ip-australia/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

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
