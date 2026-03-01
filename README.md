# MongoDB Skills

Claude Code skills that help you explore MongoDB collections, generate queries, and build institutional knowledge — all through natural language.

## What It Does

MongoDB Skills is a set of three [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) that turn Claude into a MongoDB expert for your specific database. As you ask questions, the system learns your schema, business rules, and query patterns — making each subsequent query smarter than the last.

## Skills

### `/explore-collection`

Connects to your database, samples documents, runs aggregations on every field, and builds a detailed schema file. Captures field types, value distributions, enum values, foreign key relationships, and change frequency. Supports exploring multiple collections in parallel.

```
/explore-collection visa_orders
/explore-collection users organizations countries
```

### `/write-queries`

Automatically logs every executed query with enriched metadata — pipeline stages, techniques used, fields referenced, default filters applied, and any learnings from the conversation. Runs silently after each query to build a searchable history.

### `/learn`

Processes unprocessed query logs and extracts learnings to enrich schema and guide files. Picks up field descriptions, business rules, enum values, relationships, and common patterns. Presents a summary for approval before applying changes.

```
/learn
```

## How It Works

```
  explore          query           log            learn
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Discover │──▶│ Generate │──▶│  Record  │──▶│ Enrich   │
│  schema  │   │ queries  │   │  query + │   │ schema + │
│  + types │   │  from NL │   │ metadata │   │  guide   │
└──────────┘   └──────────┘   └──────────┘   └────┬─────┘
                                                   │
                    ◀──────────────────────────────┘
                         better queries next time
```

1. **Explore** a collection to learn its schema, field types, and relationships
2. **Query** your data using natural language — Claude generates the aggregation pipeline
3. **Log** every query automatically with stages, techniques, fields, and learnings
4. **Learn** from query logs to enrich schemas and improve future query generation

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)
- [jq](https://jqlang.github.io/jq/download/) — used by explore-collection for field discovery

## Setup

1. Install the prerequisites above
2. Clone this repo
3. Create `.mcp.json` in the project root with your MongoDB connection (see template below)
4. Run `/explore-collection <collection_name>` to start building your knowledge base

### `.mcp.json` Template

Create this file in the project root (it's gitignored and will never be committed):

```json
{
  "mcpServers": {
    "mongodb": {
      "command": "npx",
      "args": ["mongodb-mcp"],
      "env": {
        "MONGODB_URI": "mongodb+srv://user:pass@cluster.mongodb.net/dbname"
      }
    }
  }
}
```

## Project Structure

```
.
├── CLAUDE.md                          # Project instructions for Claude Code
├── .claude/
│   ├── skills/
│   │   ├── explore-collection/        # Schema discovery skill
│   │   ├── write-queries/             # Query logging skill
│   │   └── learn/                     # Learning extraction skill
│   └── settings.local.json            # Tool permission rules
├── .gitignore
├── .mcp.json                          # MongoDB connection (gitignored)
└── memory/                            # All learned data (gitignored)
    ├── guide.json                     # Collection index + default filters
    ├── schema/                        # Full schema files per collection
    ├── queries/                       # Daily query logs (YYYY-MM-DD.json)
    └── learn-state.json               # Tracks which logs have been processed
```

## Memory System

The `memory/` directory (gitignored) stores everything the skills learn:

- **`guide.json`** — Quick-lookup index of all explored collections. Stores aliases, default filters, deprecated fields, foreign key references, and pointers to schema files. Read first before any query.
- **`schema/`** — One JSON file per collection with field types, presence rates, enum values, foreign keys, and user-provided notes.
- **`queries/`** — Daily query logs with enriched metadata. Each entry records the pipeline, techniques, fields used, and any learnings.

The memory directory is local to each user and not checked into version control, so each developer builds their own knowledge base tailored to their database.
