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

### `memory/schema/`
Single source of truth for all collection schemas. Stores:
- Schema definitions (field names, types, structure)
- Field aliases (e.g. `createdAt` is also referred to as "creation date")
- Foreign key relationships between collections

## MCP

The project connects to MongoDB via an MCP server configured in `.mcp.json` (gitignored — contains credentials).
