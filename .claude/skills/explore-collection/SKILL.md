---
name: explore-collection
description: Explore a MongoDB collection to learn its schema, field types, value distributions, and relationships
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task, list_collections, list_indexes, run_aggregation, run_aggregation_to_file
argument-hint: <collection_name> [collection_name2] ...
---

# Explore Collection

Explore one or more MongoDB collections to learn their schema, field types, value distributions, and relationships — then persist the results for future query generation.

**Target collection(s):** $ARGUMENTS

**Database:** Read from `memory/guide.json` → `database` field. If guide.json doesn't exist yet, ask the user which database to use.

---

## Multi-Collection Mode

If `$ARGUMENTS` contains more than one collection name (space- or comma-separated), explore **all of them in parallel**:

1. Read `memory/guide.json` and call `list_collections` once upfront. Store the database name and the full collection list.
2. For each valid collection, launch a **separate Task subagent** (subagent_type: `general-purpose`) with a prompt that contains the full single-collection exploration instructions (Steps 0–10 below), **plus the database name and the full collection list** (so the subagent skips Step 1 validation and reuses the list for foreign key checks). Run all Task calls in a **single message** so they execute in parallel.
3. After all subagents complete, read back the written schema files and present a combined summary to the user.

If only one collection is specified, follow Steps 0–10 directly (no subagent needed).

---

## Performance Mode

Track aggregation response times during exploration and adapt query behavior accordingly. This protects production clusters from heavy analytical queries.

### Modes

| Mode | `$sample` size | Parallel batch size | Restrictions |
|---|---|---|---|
| **normal** (default) | 10,000 | 5 | None |
| **light** | 1,000 | 3 | Skip array element enum detection; skip Step 7 (change frequency) |
| **minimal** | 500 | 1 (sequential) | Only analyze top-level fields (skip nested dot-notation beyond depth 2) |

### Switching Rules

- **Start in "normal" mode**
- **If any MCP aggregation response takes >30 seconds**, switch to **light** mode
- **If any MCP aggregation response takes >60 seconds**, switch to **minimal** mode
- Once you switch to a lighter mode, stay there for the rest of the exploration (do not switch back)
- After completing each batch of aggregation calls, note the current mode: `Performance mode: normal/light/minimal`

### Timing Method

Observe how long each `run_aggregation` call takes to return. You don't need precise millisecond timing — if a response is noticeably slow (long delay before result appears), treat it as exceeding the threshold.

---

## Instructions

Follow steps 0–9 in order. Use the MCP tools `list_collections` and `run_aggregation` with the database from `memory/guide.json`. Do NOT skip steps or combine aggregations in ways that could lose detail.

**Bounded aggregations:** ALL Step 6 and Step 7 aggregations MUST prepend `{ "$sample": { "size": 10000 } }` as the first pipeline stage (or the current performance mode's sample size). This avoids full collection scans. The only exception is Step 4, which uses `$sort` + `$limit` to get *recent* docs.

### Step 0: Check Existing Schema

Before doing any MCP calls, check if `memory/schema/$ARGUMENTS.json` already exists using `Glob`.

If it exists:
1. Read the file
2. Show the user a summary: document count, number of fields, when it was explored, and any notes
3. Ask: "A schema already exists for this collection (explored on {date}). Do you want to re-explore from scratch, or update the existing schema?"
4. If user says update/skip → stop here
5. If user says re-explore → continue to Step 1

### Step 1: Validate Collection

Call `list_collections` with the database from guide.json. If `$ARGUMENTS` is not in the list, show the available collections and ask the user to pick one. Do not proceed until a valid collection is confirmed.

**Important:** Store the full collection list from this call. You will reuse it in Step 6 for foreign key verification. Do NOT call `list_collections` again.

### Step 2: Fetch Indexes

Call `list_indexes` with the database and collection name. Store the raw index definitions for use in Step 9 (Write Schema File).

- Indexes reveal which fields the application considers important before sampling data
- Indexed string fields are likely high-cardinality lookup fields, not enums — use this context during field analysis (Step 6)
- Skip the default `_id` index (it's always present, no signal)

Store each index as:
- `name` — index name from MongoDB
- `key` — the key specification (field → direction/type)
- `unique` — boolean (true if unique index, false otherwise)
- Other notable properties if present: `sparse`, `ttl` (expireAfterSeconds), `partial` (partialFilterExpression)

### Step 3: Get Document Count

Run:
```json
[{ "$count": "total" }]
```

If the count is 0, write a minimal schema file (metadata only, empty fields) to `memory/schema/$ARGUMENTS.json` and stop.

Store the count as `documentCount` for later.

### Step 4: Sample Documents

Sample 100 recent documents for field discovery. Use `run_aggregation_to_file` to write results directly to a local file — zero context cost regardless of document size.

Call `run_aggregation_to_file` with:
- **pipeline:** `[{ "$sort": { "_id": -1 } }, { "$limit": 100 }]`
- **outputPath:** `/tmp/explore_{collection}.json`

The tool writes a **JSON array** to the file and returns only metadata (doc count, file path, file size). No raw documents enter the conversation context.

If the tool returns an error, stop and inform the user.

### Step 5: Build Field Inventory (via jq)

Use jq commands on the temp file to discover fields without loading documents into context.

First, verify the temp file exists and is non-empty:
```bash
test -s /tmp/explore_{collection}.json && echo "ok" || echo "EMPTY"
```
If empty/missing, stop and tell the user.

Then run these jq commands **in parallel**:

```bash
# All leaf field paths in dot-notation
jq '[.[] | [paths(scalars)] | .[] | join(".")] | unique | sort' /tmp/explore_{collection}.json

# Field types (top-level keys)
jq '[.[] | to_entries[] | {k: .key, t: (.value | type)}] | unique_by(.k)' /tmp/explore_{collection}.json

# Presence rates
jq 'length as $n | [.[] | keys[]] | group_by(.) | map({field: .[0], presence: ((length / $n) * 100 | round / 100)}) | sort_by(.field)' /tmp/explore_{collection}.json
```

Then peek at one document for orientation:
```bash
jq '.[0]' /tmp/explore_{collection}.json
```

From these results, build the field inventory:
- **type(s)** observed (string, number, boolean, date, objectId, array, object, null)
- **presence** — fraction of the 100 docs containing this field (e.g. 0.73 means 73 of 100)
- Whether it's an **array** (and what element type)
- Whether **mixed types** appear (more than one non-null type)
- Flag fields with **<10% presence** as `sparse: true`
- Flag fields that are **always null** as type `"null"` (likely deprecated)

Use dot-notation for nested fields so the final field map is flat: `address.city`, `address.zip`, etc. Cap nesting depth at 5 levels.

### Step 6: Analyze Each Field via Aggregations

For each discovered field, run targeted aggregations to understand its values. All aggregations operate on a **bounded sample** — prepend `{ "$sample": { "size": <current_mode_sample_size> } }` as the first pipeline stage.

#### Step 6a — Non-string fields (separate parallel calls, batches of 5)

For number, date, boolean, and array fields, run **separate parallel `run_aggregation` calls** — up to 5 per batch (or the current performance mode's batch size). Wait for each batch to complete before starting the next.

**Number fields** — one call per field:
```json
[
  { "$sample": { "size": 10000 } },
  { "$group": { "_id": null, "min": { "$min": "$field" }, "max": { "$max": "$field" }, "avg": { "$avg": "$field" } } }
]
```

**Date fields** — one call per field:
```json
[
  { "$sample": { "size": 10000 } },
  { "$group": { "_id": null, "min": { "$min": "$field" }, "max": { "$max": "$field" } } }
]
```

**Boolean fields** — one call per field:
```json
[
  { "$sample": { "size": 10000 } },
  { "$group": { "_id": "$field", "count": { "$sum": 1 } } }
]
```

**Array fields** — one call per field:
```json
[
  { "$sample": { "size": 10000 } },
  { "$group": { "_id": null, "minLen": { "$min": { "$size": "$field" } }, "maxLen": { "$max": { "$size": "$field" } }, "avgLen": { "$avg": { "$size": "$field" } } } }
]
```

Group related fields into batches of up to 5 calls fired in parallel. After each batch completes, note: `Performance mode: <current_mode>`.

#### Step 6b — String fields (separate parallel calls, batches of 5)

Run up to 5 string enum aggregations in parallel per round. Wait for each round before starting the next:
```json
[
  { "$sample": { "size": 10000 } },
  { "$group": { "_id": "$strField", "count": { "$sum": 1 } } },
  { "$sort": { "count": -1 } }
]
```
- If fewer than 30 distinct values → mark `isEnum: true`, store all `enumValues` with counts
- If 30+ distinct values → mark `isEnum: false`, store 5 example values

**Fire the first batch of Step 6a and the first batch of Step 6b in the same message** for maximum parallelism.

#### Foreign Key Detection

For ObjectId fields ending in `Id` or `Ref`, or fields ending in `_id` (excluding `_id` itself):
- Flag as `foreignKey` candidate
- Infer the target collection from the field name (e.g. `organizationId` → `organizations`, `user_id` → `users`)
- Verify against the collection list you cached in Step 1. Do NOT call `list_collections` again.
- Store: `{ "foreignKey": { "targetCollection": "organizations", "confirmed": true/false } }`
  - `confirmed: true` if target collection exists in the cached list
  - `confirmed: false` otherwise
  - Always use `confirmed` (boolean), never `confidence`

#### Aggregation Result Storage

**Number fields:**
```json
{ "type": "number", "presence": 1.0, "description": "...", "range": { "min": 0, "max": 100, "avg": 42.5 } }
```
Always use `range` object, never flat `min`/`max`/`avg`.

**Date fields:**
```json
{ "type": "date", "presence": 1.0, "description": "...", "range": { "min": "2024-01-01T...", "max": "2026-02-16T..." } }
```

**String fields (enum):**
```json
{ "type": "string", "presence": 1.0, "description": "...", "isEnum": true, "enumValues": { "val1": 100, "val2": 50 } }
```

**String fields (non-enum):**
```json
{ "type": "string", "presence": 1.0, "description": "...", "isEnum": false, "examples": ["val1", "val2", "val3", "val4", "val5"] }
```

**Boolean fields:**
```json
{ "type": "boolean", "presence": 1.0, "description": "...", "distribution": { "true": 800, "false": 200 } }
```

**Array fields:**
```json
{ "type": "array", "presence": 1.0, "description": "...", "arrayDetails": { "elementType": "string", "minLength": 0, "maxLength": 10, "avgLength": 3.2 } }
```
Determine element type from the sample data. If array elements are strings with <30 distinct values, unwind and count them as enum values too (skip this in **light** or **minimal** performance mode).

### Step 7: Change Frequency

**Skip this step if in light or minimal performance mode.**

Check for timestamp fields in this order:
1. `created_at` / `updated_at` (snake_case)
2. `createdAt` / `updatedAt` (camelCase)

Use whichever pair exists in the field inventory from Step 5. If neither pair exists, set `changeFrequency.hasTimestamps: false` and skip.

If a matching pair is found, run (substituting the actual field names):
```json
[
  { "$sample": { "size": 10000 } },
  { "$match": { "<updatedField>": { "$exists": true }, "<createdField>": { "$exists": true } } },
  { "$project": { "diff": { "$subtract": ["$<updatedField>", "$<createdField>"] } } },
  { "$group": {
      "_id": null,
      "avgDiff": { "$avg": "$diff" },
      "minDiff": { "$min": "$diff" },
      "maxDiff": { "$max": "$diff" },
      "docsWithUpdates": { "$sum": { "$cond": [{ "$gt": ["$diff", 0] }, 1, 0] } },
      "total": { "$sum": 1 }
    }
  }
]
```

Format `avgDiff` as human-readable (e.g. `"4d 6h 32m"`) and also store the raw millisecond value.

### Step 8: Ask User for Clarification

Present a **summary table** of all discovered fields showing: field name, type, presence %, and whether it's an enum/foreign key.

Then ask the user:
1. **Field aliases** — Are there common names or abbreviations for any fields? (e.g. `createdAt` → "creation date", `organizationId` → "org")
2. **Foreign key confirmations** — Review the detected foreign keys. Are they correct? Any missing?
3. **Deprecated fields** — Are any fields deprecated or no longer in use? (Mention any fields that are always null/empty as likely candidates.)
4. **Notes** — Any additional context? Common query patterns, business rules, soft-delete conventions, etc.

Wait for the user's response before proceeding to Step 9. If the user says "skip" or "none", proceed with empty aliases/notes.

### Step 9: Write Schema File

Assemble the final schema object and write it to `memory/schema/$ARGUMENTS.json`.

Use this exact structure:

```json
{
  "metadata": {
    "collection": "<collection_name>",
    "database": "<from guide.json>",
    "documentCount": "<number>",
    "exploredAt": "<ISO 8601 timestamp>",
    "sampleSize": 100
  },
  "indexes": [
    {
      "name": "<index_name>",
      "key": { "<field>": 1 },
      "unique": false
    }
  ],
  "fields": {
    "<fieldName>": {
      "type": "<primary_type>",
      "presence": "<0.0–1.0>",
      "description": "<auto-generated or null>",
      "...type-specific keys from Step 6"
    }
  },
  "aliases": {
    "<fieldName>": ["alias1", "alias2"]
  },
  "foreignKeys": [
    { "field": "<fieldName>", "targetCollection": "<collection>", "confirmed": true }
  ],
  "changeFrequency": {
    "hasTimestamps": true,
    "...metrics from Step 7"
  },
  "notes": ["<user-provided notes>"]
}
```

#### Auto-generated field descriptions

Instead of setting `"description": null` on every field, auto-generate descriptions for obvious fields based on name + type + values:

- `_id` → `"MongoDB document ID"`
- `__v` → `"Mongoose version key"`
- Timestamp fields (`created_at`/`createdAt`) → `"Document creation timestamp"`
- Timestamp fields (`updated_at`/`updatedAt`) → `"Last update timestamp"`
- Boolean `is_*` fields → infer from name (e.g., `is_deleted` → `"Soft delete flag"`, `is_active` → `"Active status flag"`)
- Fields with `foreignKey` → `"Reference to {targetCollection}"`
- Enum fields with short values → `"One of: {top 5 values}"`
- Set `null` only for truly ambiguous fields where the name + type don't reveal purpose

#### Key formatting rules

- Use **flat dot-notation** for nested fields (e.g. `address.city`, not nested objects)
- Mark sparse fields (<10% presence) with `"sparse": true`
- Mark mixed-type fields with `"mixedTypes": true` and list all observed types
- Mark user-confirmed deprecated fields with `"deprecated": true`
- All enum values should include their counts
- Foreign keys appear both on the field object and in the top-level `foreignKeys` array
- Always use `confirmed` (boolean) for foreign keys, never `confidence`

After writing the schema file, also update `memory/guide.json`:
1. Read `memory/guide.json` (or create it if it doesn't exist with `{ "collections": {} }`)
2. Add/update the entry under `collections.$ARGUMENTS` with:
   - `aliases` — from user notes (empty array if none)
   - `description` — from user notes (empty string if none)
   - `documentCount` — from the schema metadata
   - `defaultFilters` — from any user-specified "always filter" notes (empty object if none)
   - `deprecatedFields` — list of all fields marked `deprecated: true` in the schema
   - `references` — from the `foreignKeys` array, as a `field → targetCollection` map
   - `schemaFile` — `"memory/schema/$ARGUMENTS.json"`
   - `exploredAt` — today's date (YYYY-MM-DD)
3. Write the updated guide file back

After writing both files, print a confirmation with:
- Total fields discovered
- Number of enum fields
- Number of foreign keys detected
- File paths written

### Step 10: Cleanup

Remove the temp file:
```bash
rm -f /tmp/explore_{collection}.json
```
