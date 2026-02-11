# Mogo Skills

A set of MongoDB skills that enhance the developer experience when working with MongoDB via MCP.

## Purpose

Help developers generate MongoDB queries through the MCP server — from simple lookups to complex aggregations. Skills assist with:

- **Query generation** — build queries from natural language descriptions
- **Data analysis** — explore and understand data in collections
- **Continuous learning** — skills improve over time by recording what they learn

## Memory

Skills persist their learnings in the `memory/` folder with the following structure:

### `memory/queries/`
Day-by-day query log. Each day's queries are stored to build pattern recognition over time.

### `memory/guide.json`
Quick-lookup index of all explored collections. Stores:
- Collection names and aliases
- Default query filters (always apply these)
- Deprecated fields (never use in queries)
- Foreign key references between collections
- Pointer to full schema file

**Always read `memory/guide.json` first** before generating queries or exploring collections.

### `memory/schema/`
Single source of truth for all collection schemas. Stores:
- Schema definitions (field names, types, structure)
- Field aliases (e.g. `createdAt` is also referred to as "creation date")
- Foreign key relationships between collections

## Query Logging

Whenever you generate MongoDB queries for the user, **always** append them to the day's query log at `memory/queries/YYYY-MM-DD.json` (using today's date).

Each file is a JSON array of query entries. Append to the existing array if the file already exists, or create a new array if it doesn't.

Only log the final working query. User corrections and schema learnings go directly to `memory/schema/` files, not the query log.

Entry format:
```json
{
  "collection": "<collection_name>",
  "relatedCollections": ["<other_collection>"],
  "description": "<what the query does in plain english>",
  "pipeline": [ ... ]
}
```

- `relatedCollections` — include only when the pipeline has `$lookup` stages; lists all collections referenced. Omit for single-collection queries.

## MCP

The project connects to MongoDB via an MCP server configured in `.mcp.json` (gitignored — contains credentials).
