# @pipeworx/fr-judilibre

French judicial decisions (Cour de cassation and the judicial courts it
indexes) — search, read a decision's full pseudonymised text and structural
zones, and resolve the short codes the source uses (jurisdiction, chamber,
formation, type, solution, publication) into their French labels. Each
decision's cited texts (`visa`) come with a best-effort `{code, article}`
parse for chaining into `@pipeworx/france-legi`'s `fr_code_article`.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1856+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

Every tool needs a PISTE application key, held as a gateway platform key (never in pack source). Without one the tools say so explicitly instead of returning an empty result.

## Tools

- `fr_judilibre_search(query?, jurisdiction?, chamber?, formation?, type?, theme?, publication?, solution?, date_start?, date_end?, page?, page_size?)`
  — full-text + filtered search over the published open-data corpus. Returns
  decision ids, summaries, highlighted snippets.
- `fr_judilibre_decision(id, resolve_references?)` — one decision's full text,
  structural zones (character offsets into `text`), `visa` (cited texts) with
  `cited_articles` best-effort linked to france-legi, related decisions
  (contested/forward/rapprochements), attached files.
- `fr_judilibre_taxonomy(id?, key?, value?, context_value?)` — resolves the
  short codes the other two tools use into their French labels (or the
  reverse), or lists the taxonomy categories when `id` is omitted.

## Auth

Platform key (`PLATFORM_JUDILIBRE_KEY`, read from `_apiKey`) — not wired yet.
PISTE registration gates access beyond simple keyless signup: an application
must additionally tick the Judilibre API and accept its own CGU (separate
from the PISTE account CGU), once for sandbox and once for production — a
second click-through, not a second approval step. No identity verification,
phone, SIRET or human review was found anywhere in the flow (account
creation, application creation, or API subscription) as of 2026-10-16.

## Data sources

- Cour de cassation, Judilibre open data (judicial decisions). Served through
  the state's PISTE API gateway — never named in a tool description or
  `source` field (CLAUDE.md 9b); the route lives only in `src/index.ts`.
- Reuse: Licence Ouverte / Open Licence 2.0 (permits commercial reuse with
  attribution — the same licence already governing `@pipeworx/france-legi`),
  plus the PISTE portal's own terms of use and a Judilibre-specific terms of
  reuse document published by the Cour de cassation. The exact text of that
  last document could not be retrieved verbatim during this build (the
  courdecassation.fr page returned no extractable content); its existence and
  the licence stack above it are confirmed from the API's own OpenAPI spec
  and data.gouv.fr's dataset page.

## Implementation notes

- Auth header is `KeyId: <key>` (an OpenAPI `apiKey`-in-header scheme), not
  OAuth bearer — simpler than most PISTE APIs, which also offer an OAuth
  client id/secret.
- `visa` entries are `{ id, url, title }` with no structured code/article
  field. `cited_articles` recognises two citation phrasings actually seen in
  Cour de cassation prose ("article 1382 du code civil" and "code civil,
  art. 1382") and marks anything else `parsed: false` rather than guessing.
  The Légifrance article id embedded in `url` (`LEGIARTI…`/`LEGITEXT…`) is
  always passed through regardless of whether the title parsed.
- The published open-data corpus mirrors what the Cour de cassation selects
  for publication, not every decision ever rendered — `fr_judilibre_search`
  says so in its `completeness_note`.
- Source: Cour de cassation, Judilibre open data (judicial decisions), read per request; `data_as_of` is the date of the call.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "fr-judilibre": {
      "url": "https://gateway.pipeworx.io/fr-judilibre/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/fr-judilibre/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/fr_judilibre_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"responsabilité du fait des choses","jurisdiction":["cc"],"page_size":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/fr_judilibre_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-fr-judilibre` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/fr-judilibre/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "fr-judilibre": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-fr-judilibre"]
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
ask_pipeworx({ question: "your question about Fr Judilibre data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
