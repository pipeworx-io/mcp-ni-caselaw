# @pipeworx/ni-caselaw

Northern Ireland court and tribunal judicial decisions, read live from
judiciaryni.uk — the Court of Appeal, Crown Court, King's Bench Division,
Family Division, Coroner's Court, Master's decisions, the Valuation Tribunal
and others.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `search_ni_decisions(query?, court?, year?, limit?)` — search decisions by
  keyword, court code (e.g. `"nica"`) and/or year. Returns title, neutral
  citation, court, date, judge and a link to the decision's PDF for each match.
- `get_ni_decision(slug)` — one decision by its judiciaryni.uk slug (e.g.
  `"2026-nica-36"`). Returns full metadata + `pdf_url`; `text` is always
  `null` and `text_extracted` is always `false` — see below.

## Auth

Keyless. No operator key needed.

## Data sources

- <https://www.judiciaryni.uk/judicial-decisions> — a Drupal Views listing.
  Probed live 2026-09-18. Three real server-side facets exist:
  `/judicial-decisions/date/<year>` (also `<year>-<month>`),
  `/judicial-decisions/type/<slug-id>` and `/judicial-decisions/judiciary/<id>`,
  plus a keyword box (`?query_decisions=<text>`). **There is no court facet.**
  The court a decision belongs to is only ever visible inside its neutral
  citation and, for most modern decisions, inside its own URL slug
  (`/judicial-decisions/2026-nica-36`). `search_ni_decisions`'s `court` filter
  is therefore applied CLIENT-SIDE while paging the listing, not passed to the
  upstream as a query param — combine it with `year` to narrow a rare court
  quickly, since the per-call page budget is bounded (20 pages) and a rare
  court with no year given may need more pages than that to surface enough
  matches (the response's `truncated`/`pages_scanned` fields say so rather
  than reading as "no more decisions exist").

  `query_decisions` is a genuine full-text search across the WHOLE corpus, not
  a court filter — searching `"NICA"` returns 2,822 hits, because that literal
  string also appears inside other courts' citations of NICA cases. It is
  offered here only for keyword search, same as the site's own search box.

  `rss.xml` on this site returns 0 items — checked live, not assumed, and not
  used.

  **User-Agent matters here in an unusual way.** A UA string containing the
  word "Bot" (tested: `"...PipeworxBot/1.0..."`) gets a flat 405 from this
  site's edge (Varnish), while a plain identifying UA
  (`pipeworx-mcp-ni-caselaw/1.0 (+https://pipeworx.io)`, matching the
  monorepo's usual pattern) gets a normal 200. Not a robots.txt matter — this
  is public data with no restriction, and the ruling on robots is settled —
  just a WAF quirk worth knowing before assuming the site is blocking bots.

- **TEXT IS PDF-ONLY.** Every decision's full text is a PDF under
  `/files/judiciaryni/<yyyy-mm>/<name>.pdf`; the site's HTML never carries
  extracted text. This pack does not parse PDFs — no PDF-text-extraction
  helper exists anywhere in this monorepo (checked `mcps/*`, `shared/src`),
  and every other pack here that hits a PDF-only judicial source
  (`gao-protests`, `recap`) ships the same way: metadata + a direct link,
  `text_extracted` left `false` rather than guessed or silently empty.
  `get_ni_decision` always returns `text: null, text_extracted: false` and
  says so in its own description and in a `note` field — per the silent-zero
  policy, a caller has to be TOLD the text isn't there, never left to notice
  an empty field on its own.

- **Court codes are not a controlled vocabulary on the site** — they are
  whatever the neutral citation says. The common ones observed live: `nica`
  (Court of Appeal), `nicc` (Crown Court), `nikb` (King's Bench Division,
  called `niqb` before the 2022 accession — older decisions may use either),
  `nifam` (Family Division), `nicoroner` (Coroner's Court), `nimaster`
  (Master's decisions), `nivt` (Valuation Tribunal, whose slugs don't carry a
  year prefix, e.g. `nivt-325`). Older Social Security/Child Support
  Commissioner decisions (e.g. `"C12/25-26(PIP)"`) don't reduce to a clean
  court code at all — `deriveCourt` returns `null` for those rather than
  guessing, and a `court` filter simply won't match them (correct behaviour,
  not a bug).

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ni-caselaw": {
      "url": "https://gateway.pipeworx.io/ni-caselaw/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ni-caselaw/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/search_ni_decisions \
  -H 'Content-Type: application/json' \
  -d '{"court":"nica","year":2026,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/search_ni_decisions`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ni-caselaw": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ni-caselaw"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ni-caselaw
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ni Caselaw data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
