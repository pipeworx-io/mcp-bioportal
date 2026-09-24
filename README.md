# @pipeworx/bioportal

Term search, class lookup and cross-ontology mappings over the ~1,300
biomedical ontologies NCBO's BioPortal carries — NCIT, SNOMEDCT, LOINC, RXNORM,
MESH, HPO, ICD10CM, GO and the rest.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `bioportal_search(query, ontologies?, exact_match?, require_definitions?, limit?)`
  — free text to coded concept, with preferred label, synonyms, definition, CUI
  and the permanent class IRI.
- `bioportal_ontologies(filter?, limit?)` — the catalogue of ontologies, so you
  can find the acronym before you search.
- `bioportal_class(ontology, class_id)` — the full record for one class.
- `bioportal_mappings(ontology, class_id, to_ontology?, limit?)` — the
  equivalent concept in other ontologies, with the mapping method.

## Auth

BioPortal requires an `apikey` on every request. Resolution order in this pack:

1. caller-supplied `_apiKey` (free key, https://bioportal.bioontology.org/accounts/new),
2. otherwise the demo key NCBO publishes in its own REST documentation, which is
   shared across all of its users and rate-limited accordingly.

Every response carries `key_source` so a caller can see which one answered.

**When a platform key is provisioned** (`PLATFORM_BIOPORTAL_KEY`), the change is
one line: add `"platformKeyEnv": "PLATFORM_BIOPORTAL_KEY"` to this pack's entry
in `workers/gateway/src/pack-manifest.json` and re-run
`node scripts/sync-pack-manifest.mjs`. The gateway then injects it as `_apiKey`
and the demo fallback stops being reached. It is deliberately NOT declared
today: `keyBlockedTools()` in the gateway marks every tool of a pack that
declares an UNSET `platformKeyEnv` as key-blocked, which would sink a pack that
currently works for everyone.

## Data sources

- <https://data.bioontology.org/search> — `q`, `ontologies`, `exact_match`,
  `require_definitions`, `pagesize`.
- <https://data.bioontology.org/ontologies> — the catalogue.
- <https://data.bioontology.org/ontologies/{acronym}/classes/{URL-encoded IRI}>
  and its `/mappings` child.

Things that will otherwise cost you an afternoon:

- **`display_links=false` is the difference between 4.4 MB and 312 KB** on
  `/ontologies`. Use it there.
- **But do NOT set it on `/search`**: a search hit's ontology acronym is only
  derivable from `links.ontology`. Its `@id` is a purl
  (`http://purl.bioontology.org/ontology/MESH/D008545`) whose path segment is
  not reliably the acronym.
- **The class IRI must be URL-encoded as a single path segment** — `#` and `/`
  included — e.g.
  `/ontologies/NCIT/classes/http%3A%2F%2Fncicb.nci.nih.gov%2Fxml%2Fowl%2FEVS%2FThesaurus.owl%23C3224`.
- **A mapping record lists BOTH ends in `classes`**, including the class you
  asked about. Pick the one whose ontology is not the source, or every mapping
  looks like a self-mapping.
- **Not every concept maps to the big terminologies.** NCIT "Melanoma" (C3224)
  has 114 mappings and none of them are to SNOMEDCT — the targets are CADSR-VS,
  RH-MESH, MESH, MEDDRA, LOINC and a long tail. An empty `to_ontology` filter
  result is a real answer, not a failure.

## Scope, so nobody builds this twice

The `ols` pack covers EBI's Ontology Lookup Service — a different repository
with a different, OBO-leaning set of ontologies. BioPortal is the one carrying
the US clinical terminologies. `cbioportal` is cancer genomics and is unrelated
despite the name.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bioportal": {
      "url": "https://gateway.pipeworx.io/bioportal/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bioportal/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/bioportal_search \
  -H 'Content-Type: application/json' \
  -d '{"query":"melanoma","ontologies":"NCIT,DOID","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bioportal_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bioportal": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bioportal"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bioportal
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bioportal data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
