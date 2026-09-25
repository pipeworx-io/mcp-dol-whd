# @pipeworx/dol-whd — DOL Wage & Hour enforcement

Concluded US Department of Labor **Wage & Hour Division** compliance actions since
FY2005 — wage theft, back wages, minimum-wage and overtime findings, child labour,
H-1B and H-2A violations. `osha` already covers the OSHA slice of DOL enforcement;
this is the sibling nobody could query.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

Promoted out of `_incubator` on 2026-09-01 (fleet #1052) once Bruce supplied
the key. Built originally against a 10-row catalog preview under fleet #651,
because until that key existed every row request answered 401.

## Tools

| Tool | Answers |
|---|---|
| `whd_search` | "Has *employer* been cited for wage violations?" — filter by employer, NAICS, state, date |
| `whd_employer` | Full case history and aggregate totals for one employer |
| `whd_top_backwages` | "Who owes the most back wages in *industry*?" |
| `whd_coverage` | What the dataset covers, its publication date and findings-date range |

## Auth

Platform key (`PLATFORM_DOL_KEY`, set on gateway and registry-api) with BYO
override via `_apiKey`. A free key from <https://dataportal.dol.gov> covers every
DOL dataset and works on issue — there is no approval step. `_apiKey` is **not**
in any tool's `required` list, because the gateway injects the platform key.

This is **not** the data.gov umbrella key ([[reference-datagov-key-universal]]
does not apply); the DOL Open Data Portal issues its own.

**The catalog is open and the rows are not, which is a trap.** `GET /v4/datasets`
and `GET /v4/datasets/10362` both answer **200 with no key**, so the source looks
open until you ask for data — at which point a keyless row request returns **401**
`"The API key is either incorrect or missing from your query."`

**Worse, auth differs by endpoint.** `/v4/datasets*` accepts an `X-API-KEY:`
request **header**; `/v4/get/...` **rejects** it with 401 and needs the key as a
**query parameter**. Standardise on the header because the catalog call worked
and every data call 401s, which reads as a bad key rather than a wrong place to
put it. The pack always uses the query parameter.

**The upstream throttles per SOURCE IP, not per key.** Roughly 20 calls in a few
minutes returns HTTP 429 with `x-amzn-errortype: ForbiddenException` and no
`Retry-After`, for about ten minutes. Measured: a *bogus* key from a throttled IP
also gets 429 while omitting the key entirely still gets 401, which is what pins
it to the IP. So this is a shared-egress hazard for us rather than a per-key
quota. The pack backs off twice and then returns
`{ found: false, reason: "upstream_rate_limited" }` rather than throwing, so a
throttle stays distinguishable from an employer with no cases.

## Data sources

- <https://apiprod.dol.gov/v4/get/whd/enforcement/json> — case rows (key required)
- <https://apiprod.dol.gov/v4/datasets/10362> — dataset metadata + 10-row preview (open)
- <https://dataportal.dol.gov> — key registration and the API Query Builder

## Query grammar (documented, not guessed)

`data.dol.gov` is a React SPA whose bundle `/static/js/main.1788ccf8.js` carries
DOL's own API documentation. Parameters: `limit`, `offset`, `sort` (`asc`/`desc`),
`sort_by` (field name), `fields`, `filter_object`, `X-API-KEY` — all as query
parameters. Operators, quoted from that documentation: **`eq`, `neq`, `gt`, `lt`,
`in`, `not_in`, `like`**. Nothing outside that list is assumed to work.

`filter_object` is a JSON string of nested `and` / `or` groups over
`{field, operator, value}`, per DOL's worked example:

```
filter_object={"and":[{"or":[{"field":"industry","operator":"eq","value":"A"},
                             {"field":"industry","operator":"eq","value":"C"}]},
                      {"field":"year","operator":"eq","value":"2021"}]}
```

## Traps this pack already handles

- **A zero-match query answers HTTP 204 with an empty body**, not a 200 with
  `{"data":[]}`. `res.ok` is true for 204, so parsing without checking the status
  throws on empty input — turning "this employer has no cases", the commonest
  negative result there is, into what looks like a parse bug of ours.
- **`filter_object` keywords must be lowercase** — `field`, `operator`, `value`,
  `and`, `or`. Anything else answers **500** with a generic "check for typos"
  message that names no field. A wrong *dataset* name gives the same 500, which
  is why `whd/whisard` (the name in the original brief) read as a DOL outage
  rather than a typo. The dataset is `whd/enforcement`.
- **`like` needs `%` to behave as a substring match.** DOL documents `like` as
  the substring operator but not its syntax. Measured 2026-09-01: `%Walmart%`
  and bare `Walmart` both return rows, but the wildcard form is a strict
  superset — it also matches "Subway Georgetown Walmart", which the bare form
  does not. Bare `like` is an anchored match. Every employer search here uses
  `%term%`.
- **Trade name ≠ legal name.** "Reliant Energy" vs "Reliant Energy Retail
  Services, LLC" are the same employer. Both are matched with `like`, and every
  hit carries `matched_field` saying which one landed.
- **`bw_atp_amt` is the TOTAL, and the FLSA columns are a trap.** The
  statute-level columns (`flsa_bw_atp_amt`, `sca_bw_atp_amt`, `cwhssa_bw_amt`,
  `dbra_…`, `mspa_…`, `h1b_…`, `fmla_…` …) are mutually exclusive and sum to
  `bw_atp_amt` — verified on cases with two statutes (Corrections Corp of
  America: 7,118,609 SCA + 953,252 CWHSSA = 8,071,861; Hewlett-Packard:
  4,831,719 SCA + 401,211 FLSA = 5,232,930, both exact). But `flsa_ot_…`,
  `flsa_mw_…`, `flsa_15a3_…` and the `flsa_smw*` set are a **breakdown of**
  `flsa_bw_atp_amt`, not siblings of it, so adding all the `*_bw_*_amt` columns
  double-counts. The pack returns `back_wages_agreed_usd` (the total),
  `back_wages_by_statute`, and `flsa_breakdown` separately, plus a
  `back_wages_reconciles` flag so a change in that structure surfaces in a
  response rather than silently inflating a number.
- **Findings dates are not case open/close dates.** The dataset says so
  explicitly and does not carry the latter. Register freshness on a findings date.
- **But do not take the newest findings date naively.** Rows with a NULL
  `findings_end_date` sort to the top of a `sort=desc`, so the obvious query
  returns `null`; and DOL's own data carries typo'd future dates — the newest
  non-null row is stamped **3021-05-01**. `whd_coverage` bounds the window at
  both ends and says so in `window_note`.
- **`www.dol.gov` HTML pages 403 a plain UA behind Akamai** and need the full
  browser header set (`BROWSER_HEADERS` in
  `workers/data-pipeline/src/datasets/dod-contracts.ts`). That is a
  header-fingerprint block, **not** a CF-egress block — do not build a proxy for
  it. `apiprod.dol.gov` is a separate host and shows no such gating.

## Resolved on promotion

- **Is `bw_atp_amt` the total of the act columns, or a separate bucket?**
  It is the **total** of the statute-level columns — settled on real rows, see
  the trap above. This was the pack's one open question while it had no key.
- **Reachable from a deployed Cloudflare Worker?** Yes — verified live through
  the gateway on 2026-09-01. `apiprod.dol.gov` does not gate CF egress. (The
  `www.dol.gov` Akamai block noted above is a different host and still applies
  to HTML scraping.)
- **`PLATFORM_DOL_KEY`** is set as a wrangler secret on both `pipeworx-gateway`
  and `pipeworx-registry-api`.

## The other 41 DOL datasets

The same v4 API serves 42 datasets across nine agencies (MSHA 15, OSHA 11,
ILAB 7, ETA 4, and one each for EBSA, TRNG, VETS, WB and WHD). The full list,
with the call shape and every trap above, is recorded in
[`docs/dol-open-data-catalog.md`](../../docs/dol-open-data-catalog.md).
**Survey only** — nothing there is built, and post-saturation nothing should be
built from it speculatively; it exists so a demand signal has somewhere to land.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "dol-whd": {
      "url": "https://gateway.pipeworx.io/dol-whd/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/dol-whd/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/whd_search \
  -H 'Content-Type: application/json' \
  -d '{"employer":"Walmart"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/whd_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "dol-whd": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-dol-whd"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-dol-whd
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Dol Whd data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
