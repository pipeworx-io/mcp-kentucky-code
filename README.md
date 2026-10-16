# @pipeworx/kentucky-code

Kentucky Revised Statutes (KRS) — state statutes by citation and by topic
search, with legislative/amendment history. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1819+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `ky_statute(citation)` — the current text of a KRS section by citation
  (`"507.020"`, `"KRS 383.580"`, `"§ 507.020"`). Returns the catchline, the
  full statutory text, the `effective_date`, and the `history` line (the
  Acts that enacted and amended it). A prior (pre-amendment) version's text
  is not available anywhere online — `historical_versions` says so rather
  than silently omitting it.
- `ky_search(query, limit?)` — full-text topic search over the statutory
  body text, not just catchlines (`"landlord"` surfaces KRS 383.580
  "Security deposits", a catchline that contains neither word). Returns
  matching citations and catchlines; pass a citation to `ky_statute` for the
  full text.

## Auth

Keyless. No signup, no key, on either side of the call.

## Data sources

- <https://apps.legislature.ky.gov/LRCSearch/Home/getStatuteJSON> — the
  Legislative Research Commission's own citation/topic search, GET with
  `searchPhrase`, `statuteRange=true&startRange=1&endRange=999`, and
  `searchStatuteNumber=true` for an exact-citation lookup. Needs a fresh
  `RequestVerificationToken` header, read from an inline `<script>` on
  <https://apps.legislature.ky.gov/lrcsearch> — NOT cookie-bound (confirmed:
  the search page sets no cookie), so one GET for the token followed by one
  GET against the API, per call, is the whole mechanism. No login.
- <https://apps.legislature.ky.gov/law/statutes/statute.aspx?id=> — the
  matched section's full text, served as a per-section PDF
  (`content-disposition: inline;FileName="KRS507_020(K).pdf"`). There is no
  HTML page with the statutory text anywhere on the site; the chapter index
  pages list only catchlines. Extracted via the new `extractPdfText` helper
  in `shared/src/pdf-text.ts` (see that file for what it does and does not
  handle) — this repo had no PDF text extraction before this pack.

**The opaque-id trap, and why this isn't the two-hop resolver.** The August
state-law survey (`docs/state-law-probe.md`) flagged Kentucky's statute URLs
as addressed by an opaque database id with no derivable mapping from the
citation — the same shape Vermont and Connecticut solved with a two-hop fetch
through the chapter index. Kentucky's chapter index pages (`chapter.aspx?id=`)
DO carry that mapping (confirmed: Chapter 507's index lists ".020 Murder." next
to its id), so the two-hop route would work — but a simpler ONE-hop route also
works and is what this pack uses: the LRC's own search API resolves a citation
straight to its id, because it is a real full-text search over the whole KRS
rather than a title/catchline index (verified live: searching "landlord"
surfaces KRS 383.580, whose catchline is just "Security deposits.").

**Four capabilities, three available.** Citation lookup and topic search are
both the same search endpoint with a flag; amendments/history is the
statute's own `History:` line, parsed out of the extracted PDF text. The
fourth — the TEXT of a section as it read before a given amendment — is not
published online anywhere: a superseded id redirects to an "Invalid Statute"
page instructing the reader to e-mail the Reviser of Statutes for it. This
pack reports that as `historical_versions`, not as a silent gap.

**Division/chapter-heading rows.** The search API's `Type` field is `"K"` for
a real statute section, `"D"` for a chapter/division heading (no statutory
text — `ky_statute` reports `not_a_section` rather than fetching a PDF for
it), and `"R"` for a repealed section (text still returned; `type_label`
says which).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "kentucky-code": {
      "url": "https://gateway.pipeworx.io/kentucky-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/kentucky-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1819+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ky_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"507.020"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ky_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "kentucky-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-kentucky-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-kentucky-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Kentucky Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
