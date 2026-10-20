# realm-graphiti-memory

A self-hosted [Graphiti](https://github.com/getzep/graphiti) server's temporal knowledge graph
under the shared `Memory` label. Graphiti — the engine behind Zep — extracts facts from what it is
told and keeps, for each, the window in which it held. This realm recalls those facts through the
server's own hybrid search, so one Virtual Cypher query answers from Graphiti alongside the
assistant's own memory and every other memory system.

```cypher
MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:Memory)          -- every memory system
RETURN m.source, m.text, m.validFrom, m.validTo, r.score ORDER BY m.source

MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:GraphitiMemory)  -- Graphiti alone
RETURN m.text, m.name, m.validFrom, m.validTo ORDER BY r.rank

MATCH (:Concept {value:'Ada'})-[:RELEVANT_TO]->(m:GraphitiMemory {status:'SUPERSEDED'})
RETURN m.text, m.validFrom, m.validTo                                  -- what no longer holds
```

Every node carries `:Memory`, `:GraphitiMemory` and the realm label `:Graphiti`.

## What it needs

A Graphiti server — the FastAPI service the graphiti repo ships, image `zepai/graphiti` — with
its Neo4j (or FalkorDB) and an `OPENAI_API_KEY`. The realm's vendored OpenAPI document points at
`http://127.0.0.1:8000`; change `servers[0].url` in `apis/graphiti-openapi.json` for yours. The
reference server has no authentication, so keep it beside the assistant, not on a network.

`group_ids` is left unset in `producers/graphiti.yml`, which searches every group the server
holds. A server shared between users must be given the acting user's group there.

## The contract

| Memory property | From Graphiti |
|---|---|
| `id` | the fact's `uuid` |
| `text` | `fact` |
| `name` | the relation name (`DRINKS_AT`, `WORKS_AT`, …) |
| `status` | `ACTIVE` while the fact holds; `SUPERSEDED` once `invalid_at` or `expired_at` is set |
| `validFrom` / `validTo` | `valid_at` / `invalid_at` |
| `created` / `revised` | `created_at` / `expired_at` |
| `episodes` | the episodes the fact was extracted from |
| `r.score` | 1.0 — the server's search returns no score; `r.rank` is its order |

A plain read returns ACTIVE facts; pin `{status:'SUPERSEDED'}` on the node to read what Graphiti
has invalidated. That is the `Memory` lifecycle, declared once in the default world and inherited.

## Zep Cloud

Zep's hosted service is the same engine with a key and a per-user graph: see `realm-zep-memory`.

## Known server behaviour

The reference server's `POST /messages` queues ingestion to a background worker that, in the
image tested (`zepai/graphiti:latest`, 2026-10), fails with "Driver closed" once the request has
returned. Ingest with `graphiti-core` directly against the same database, or through Graphiti's
MCP server; the realm only reads.
