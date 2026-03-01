---
name: explore-collection
description: Explore a MongoDB collection to learn its schema, field types, value distributions, and relationships
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Task, list_collections, run_aggregation, run_aggregation_to_file
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
2. For each valid collection, launch a **separate Task subagent** (subagent_type: `general-purpose`) with a prompt that contains the full single-collection exploration instructions (Steps 0–9 below), **plus the database name and the full collection list** (so the subagent skips Step 1 validation and reuses the list for foreign key checks). Run all Task calls in a **single message** so they execute in parallel.
3. After all subagents complete, read back the written schema files and present a combined summary to the user.

If only one collection is specified, follow Steps 0–9 directly (no subagent needed).

---

## Reference: $facet Guidelines

`$facet` lets you run multiple sub-pipelines in a single aggregation call, reducing round trips. But it has tradeoffs:

**How `$facet` works:** Each sub-pipeline runs sequentially against the same input documents. It does NOT parallelize. So 5 heavy sub-pipelines = 5 sequential scans of the same data.

**Use `$facet` when:**
- The per-facet pipelines are lightweight (single `$group` stage)
- You're combining related analyses (e.g., all number field ranges in one call)
- Max 5 facets per call to keep response time reasonable

**Do NOT use `$facet` when:**
- A sub-pipeline uses `$unwind`, `$lookup`, or other expensive stages — run these separately
- You have more than 5 sub-pipelines — split into multiple `$facet` calls
- The collection has `documentCount > 500,000` AND you're not using `$limit` — always add `$sort` + `$limit` first on large collections

**Prefer separate parallel calls when:**
- String enum detection — these are heavier (group + sort). Run up to 5 separate string aggregations in parallel per round rather than putting them in `$facet`
- You're unsure about the cost — separate calls are always safe; `$facet` is an optimization

---

## Instructions

Follow steps 0–9 in order. Use the MCP tools `list_collections` and `run_aggregation` with the database from `memory/guide.json`. Do NOT skip steps or combine aggregations in ways that could lose detail.

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

**Important:** Store the full collection list from this call. You will reuse it in Step 5 for foreign key verification. Do NOT call `list_collections` again.

### Step 2: Get Document Count

Run:
```json
[{ "$count": "total" }]
```

If the count is 0, write a minimal schema file (metadata only, empty fields) to `memory/schema/$ARGUMENTS.json` and stop.

Store the count as `documentCount` for later.

### Step 3: Sample Documents

Sample 100 recent documents for field discovery. Use `run_aggregation_to_file` to write results directly to a local file — zero context cost regardless of document size.

Call `run_aggregation_to_file` with:
- **pipeline:** `[{ "$sort": { "_id": -1 } }, { "$limit": 100 }]`
- **outputPath:** `/tmp/explore_{collection}.json`

The tool writes a **JSON array** to the file and returns only metadata (doc count, file path, file size). No raw documents enter the conversation context.

If the tool returns an error, stop and inform the user.

### Step 4: Build Field Inventory (via jq)

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

### Step 5: Analyze Each Field via Aggregations

For each discovered field, run targeted aggregations to understand its values. Use the **full collection** (not just the sample) for these aggregations.

**For collections with `documentCount > 500,000`:** Prepend `{ "$sort": { "_id": -1 } }, { "$limit": 10000 }` to every aggregation pipeline (both `$facet` and separate calls) to analyze the latest 10K documents.

#### Step 5a — Non-string fields (one or two `$facet` calls)

Combine all number, date, boolean, and array field analyses into `$facet` calls (max 5 facets each):

```json
[{ "$facet": {
  "numbers": [{ "$group": { "_id": null, "field1_min": {"$min": "$field1"}, "field1_max": {"$max": "$field1"}, "field1_avg": {"$avg": "$field1"} }}],
  "dates": [{ "$group": { "_id": null, "field2_min": {"$min": "$field2"}, "field2_max": {"$max": "$field2"} }}],
  "bool_field1": [{ "$group": { "_id": "$bool_field1", "count": {"$sum": 1} }}],
  "array_field1": [{ "$group": { "_id": null, "minLen": {"$min": {"$size": "$arr1"}}, "maxLen": {"$max": {"$size": "$arr1"}}, "avgLen": {"$avg": {"$size": "$arr1"}} }}]
}}]
```

If there are more than 5 facets, split into multiple `$facet` calls.

#### Step 5b — String fields (separate parallel calls, batches of 5)

Run up to 5 string enum aggregations in parallel per round. Wait for each round before starting the next:
```json
[{ "$group": { "_id": "$strField", "count": { "$sum": 1 } } }, { "$sort": { "count": -1 } }]
```
- If fewer than 30 distinct values → mark `isEnum: true`, store all `enumValues` with counts
- If 30+ distinct values → mark `isEnum: false`, store 5 example values

**Fire Step 5a and the first batch of Step 5b in the same message** for maximum parallelism.

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
Determine element type from the sample data. If array elements are strings with <30 distinct values, unwind and count them as enum values too.

### Step 6: Change Frequency

Check for timestamp fields in this order:
1. `created_at` / `updated_at` (snake_case)
2. `createdAt` / `updatedAt` (camelCase)

Use whichever pair exists in the field inventory from Step 4. If neither pair exists, set `changeFrequency.hasTimestamps: false` and skip.

If a matching pair is found, run (substituting the actual field names):
```json
[
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

### Step 7: Ask User for Clarification

Present a **summary table** of all discovered fields showing: field name, type, presence %, and whether it's an enum/foreign key.

Then ask the user:
1. **Field aliases** — Are there common names or abbreviations for any fields? (e.g. `createdAt` → "creation date", `organizationId` → "org")
2. **Foreign key confirmations** — Review the detected foreign keys. Are they correct? Any missing?
3. **Deprecated fields** — Are any fields deprecated or no longer in use? (Mention any fields that are always null/empty as likely candidates.)
4. **Notes** — Any additional context? Common query patterns, business rules, soft-delete conventions, etc.

Wait for the user's response before proceeding to Step 8. If the user says "skip" or "none", proceed with empty aliases/notes.

### Step 8: Write Schema File

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
  "fields": {
    "<fieldName>": {
      "type": "<primary_type>",
      "presence": "<0.0–1.0>",
      "description": "<auto-generated or null>",
      "...type-specific keys from Step 5"
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
    "...metrics from Step 6"
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

### Step 9: Cleanup

Remove the temp file:
```bash
rm -f /tmp/explore_{collection}.json
```
