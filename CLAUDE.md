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

## Query Logging (MANDATORY)

**Every MongoDB query you run MUST be logged immediately after execution.** Use the `write-queries` skill for all query logging. Do not wait for the user to remind you — log queries as part of the same response where you run them.

Log to `memory/queries/YYYY-MM-DD.json` (today's date). Append to the existing array if the file exists, or create a new array if it doesn't.

Rules:
- Log every final working query in the same turn you execute it
- Only log the final working query — not intermediate/failed attempts
- Learnings (corrections, business rules, field discoveries) are captured in the query log's `learnings` field — the skill handles this automatically

The `write-queries` skill enriches each log entry with: `stages`, `techniques`, `fieldsUsed`, `defaultFiltersApplied`, `tags`, and `learnings`. See the skill definition for the full schema.

## MCP

The project connects to MongoDB via an MCP server configured in `.mcp.json` (gitignored — contains credentials).
