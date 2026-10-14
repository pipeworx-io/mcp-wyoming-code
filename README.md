# Wyoming Statutes (W.S.)

Wyoming Statutes by citation, plus full-text topic search. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1800+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Source (fleet #2744)

The Wyoming Legislative Service Office (wyoleg.gov) runs an Angular SPA for
its "Wyoming Statutes & Constitution" page. Probed live 2026-10-06 by
downloading the SPA's own JS bundles (`main-*.js`, the lazily-loaded
`chunk-_lXRwsur.js` for its StateStatutesModule) and grepping for an API base
URL or any `lsoservice.wyoleg.gov` reference — none exists for statute TEXT.
The "Download the Wyoming Statutes" page is a plain static list of links to
per-TITLE PDFs:

```
https://wyoleg.gov/statutes/compress/title06.pdf   (Title 6, Crimes and Offenses)
```

A `.docx` sibling (the task's other named possibility) does NOT exist — every
`.docx` path probed (several casings and locations) returns the SPA's own
catch-all shell (HTTP 200, not a real 404), confirmed by checking
content-type/length rather than status code. PDF is the only route.

## The PDF trap, and why `shared/src/pdf-text.ts` needed extending

Wyoming's per-title PDFs are PDF 1.6, linearized, and store their Catalog and
Pages root inside a **compressed object stream** (`/Type/ObjStm`) — a shape
`shared/src/pdf-text.ts` explicitly did not support before this pack (its own
header said so; Kentucky's PDFs, the only prior user, never hit it). Verified
directly against Title 6's PDF: `5205 0 obj` (its Pages root) never appears
literally in the file at all; it exists only as an entry inside a decompressed
ObjStm. Separately, the Catalog object IS direct but the original discovery
regex broke the moment a nested dict (`/MarkInfo<</Marked true>>`) appeared
before `/Type/Catalog` in the same object — also true here, not Wyoming-
specific bad luck.

Both gaps are now fixed in `shared/src/pdf-text.ts` itself (see that file's
header for the mechanism), not worked around in this pack, since a future
PDF-only state with a compressing producer hits the identical wall. Verified
end-to-end: Title 6 (441 KB, 210 pages) extracts to 387,932 characters with
zero warnings, and W.S. 6-2-101 ("Murder in the first degree") comes back with
its correct text.

## MB-scale files — never fetched whole per call

Per-title PDFs run from ~300 KB (Title 97, the Constitution — not covered, see
below) to ~2.9 MB (Title 35, Public Health and Safety) across the 44 titles
this pack covers. Exactly like `mcps/indiana-code`, one JSON bundle per title
is pre-fetched into the shared `pipeworx-datasets` R2 bucket
(`statutes/wy/<title>.json`, via `scripts/ingest-wyoming-statutes.mjs`) plus a
full-text search shard (`wy-statutes`). `wy_statute` reads the pre-sliced
section straight out of the bundle; a live PDF fetch is the FALLBACK only —
for a title the bundle does not yet cover, or a standalone (non-hosted)
install with no R2 binding — and even then fetches only the ONE requested
title's PDF, never the whole corpus.

## Tools

- `wy_statute(citation)` — full text of a Wyoming Statutes section by
  citation ("6-2-101" is murder in the first degree, Title 6; "20-1-105" is
  divorce grounds, Title 20). Accepts "6-2-101", "W.S. 6-2-101", "§ 6-2-101".
  Returns the current text, the section heading, and — only where the source
  carries one cleanly — an enactment `history` note (see below).
- `wy_search(query, limit?)` — full-text search over the statutory body text
  across all 44 covered titles via the shared FTS5 search shard. Returns
  matching citations and headings; call `wy_statute` for the full text.

## Citations

Wyoming cites title-chapter-section, dash separated, e.g. "6-2-101" or
"20-1-105". A few titles carry a decimal number (e.g. "34.1-1-101", the
Uniform Commercial Code). Title numbers map directly to the PDF filename
(`title06.pdf`, `title34.1.pdf`, `title99.pdf` — single-digit titles are
zero-padded, everything else is used as-is).

## Slicing a section out of a whole title

Wyoming's extracted PDF text has no HTML-style structural markers (unlike
Indiana's `<div class="section" id="...">`) — it is one long run per page,
because the content streams use relative `Td`/`T*` text positioning rather
than the absolute `Tm` this repo's PDF extractor tracks for line breaks. The
citation itself is what locates a section: every real heading is typeset
"N-N-N.  Caption text." (a two-space sentence gap, confirmed in the raw
content stream and used consistently throughout). A plain citation match is
not enough on its own, though — a cross-reference ("...sentenced in
accordance with the provisions of W.S. 6-2-102.") uses the exact same
two-space convention after ITS OWN trailing period, so a naive match finds
both the real heading and every inline cross-reference to the same number.

This pack uses the shared `sliceSection` "most text wins" heuristic
(`shared/src/statute-html.ts`, already relied on by the Colorado/Iowa/
Nevada/Oregon pack family) rather than inventing a Wyoming-specific one:
among every literal occurrence of the target citation followed by two spaces,
the real heading is whichever has the most text before the next
citation-shaped boundary. Proven on 6-2-101 specifically — its only other
occurrence in Title 6 is a cross-reference that sits one sentence before the
immediately next real section (6-2-202), so it resolves to near-zero
trailing text and loses decisively. The ingest script applies the same
"longest wins" principle in one pass across a whole title (grouping every
boundary by citation) rather than per-citation, since a full crawl has no
single "target" to search for.

## Four capabilities — two available, two are not

- **Citation lookup** — available, `wy_statute`.
- **Topic/full-text search** — available, `wy_search`.
- **Amendments/enactment history** — BEST EFFORT ONLY. Some titles (e.g. 99,
  Noncodified Statutes — mostly older uniform-act material) carry a clean
  trailing `(Laws 1985, ch. ...)` parenthetical per section, which
  `wy_statute` pulls into a separate `history` field when present. Title 6
  (the current recodified criminal code) carries NO such trailing block —
  amendment/repeal notes appear INLINE at the subsection level ("(E)
  Repealed by Laws 2020, ch. 90, § 3.") as ordinary statutory text, with no
  structural marker separating them. `history` is `null` on most sections;
  that is not an extraction failure.
- **Historical version** (the text of a section before a given amendment, or
  a prior year's edition) — NOT AVAILABLE. wyoleg.gov's statutes page serves
  only the current edition ("as they exist as of July 1, 2026" per its own
  download page) — no year parameter, no archive link. Confirmed by grepping
  the SPA bundle for any "historical"/"archive" reference: none exists.
  `wy_statute` says so explicitly in its response rather than omitting the
  field silently.

## Out of scope

Title 97 (the Wyoming Constitution) cites "Article N, Section N" rather than
"N-N-N" — a different grammar entirely — and is excluded from this pack's
citation parsing and from the ingest script's title list. A Wyoming
Constitution pack, if built, is a separate source. Title 99 (Noncodified
Statutes) DOES use the standard "N-N-N" scheme and IS included.

## Coverage

All 44 titles that use the title-chapter-section citation scheme: 1-42, the
decimal 34.1 (Uniform Commercial Code), and 99 (Noncodified Statutes).
21,268 sections ingested.

## Refresh

`node scripts/ingest-wyoming-statutes.mjs` — re-run periodically (the
Legislature convenes annually; no daily-freshness need). No schedule is wired
up: dispatch by hand, following `scripts/ingest-indiana-statutes.mjs`'s
pattern. PDF extraction is duplicated as plain JS in `scripts/lib/pdf-text.mjs`
(a deliberate line-for-line port of `shared/src/pdf-text.ts`'s
`extractPdfText`) because `scripts/` cannot resolve a bare `@pipeworx/shared`
specifier outside the pnpm workspace's own per-package `node_modules` link —
see that file's header.

## Data source

Official state legislature site (wyoleg.gov). Wyoming statutes are public
record; no reuse restriction was found on the per-title PDF paths used here.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "wyoming-code": {
      "url": "https://gateway.pipeworx.io/wyoming-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/wyoming-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1800+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/wy_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"6-2-101"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/wy_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "wyoming-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-wyoming-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-wyoming-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Wyoming Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
