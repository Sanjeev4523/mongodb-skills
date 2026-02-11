---
name: explore-collection
description: Explore a MongoDB collection to learn its schema, field types, value distributions, and relationships
allowed-tools: Read, Write, Edit, Glob, Grep, list_collections, run_aggregation
argument-hint: <collection_name>
---

# Explore Collection

Explore a MongoDB collection to learn its schema, field types, value distributions, and relationships — then persist the results for future query generation.

**Target collection:** $ARGUMENTS

**Database:** Read from `memory/guide.json` → `database` field. If guide.json doesn't exist yet, ask the user which database to use.

---

## Instructions

Follow steps 0–8 in order. Use the MCP tools `list_collections` and `run_aggregation` with the database from `memory/guide.json`. Do NOT skip steps or combine aggregations in ways that could lose detail.

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

### Step 2: Get Document Count

Run:
```json
[{ "$count": "total" }]
```

If the count is 0, write a minimal schema file (metadata only, empty fields) to `memory/schema/$ARGUMENTS.json` and stop.

Store the count as `documentCount` for later.

### Step 3: Sample Latest 100 Documents

Run:
```json
[{ "$sort": { "_id": -1 } }, { "$limit": 100 }]
```

These 100 documents are your **sample set**. All field discovery in Step 4 comes from this sample.

### Step 4: Build Field Inventory

Analyze the 100 sampled documents to discover every field, including nested fields using dot-notation (e.g. `address.city`). Cap nesting depth at 5 levels — if deeper nesting exists, note it but don't traverse further.

For each field record:
- **type(s)** observed (string, number, boolean, date, objectId, array, object, null)
- **presence** — fraction of the 100 docs containing this field (e.g. 0.87 means 87 of 100)
- Whether it's an **array** (and what element type)
- Whether **mixed types** appear (more than one non-null type)
- Flag fields with **<10% presence** as `sparse: true`
- Flag fields that are **always null** as type `"null"` (likely deprecated)

Use dot-notation for nested fields so the final field map is flat: `address.city`, `address.zip`, etc.

### Step 5: Analyze Each Field via Aggregations

For each discovered field, run targeted aggregations to understand its values. Use the **full collection** (not just the sample) for these aggregations. Add `{ "$limit": 10000 }` as the first pipeline stage on collections with `documentCount > 1000000`.

**Strings:**
```json
[{ "$group": { "_id": "$fieldName", "count": { "$sum": 1 } } }, { "$sort": { "count": -1 } }]
```
- If fewer than 30 distinct values → mark `isEnum: true`, store all `enumValues` with counts
- If 30+ distinct values → mark `isEnum: false`, store 5 example values

**Numbers:**
```json
[{ "$group": { "_id": null, "min": { "$min": "$fieldName" }, "max": { "$max": "$fieldName" }, "avg": { "$avg": "$fieldName" } } }]
```
Store as `range: { min, max, avg }`.

**Dates:**
```json
[{ "$group": { "_id": null, "min": { "$min": "$fieldName" }, "max": { "$max": "$fieldName" } } }]
```
Store as `range: { min, max }`.

**Booleans:**
```json
[{ "$group": { "_id": "$fieldName", "count": { "$sum": 1 } } }]
```
Store as `distribution: { "true": N, "false": M }`.

**ObjectIds ending in `Id` or `Ref`:**
Flag as `foreignKey` candidate. Infer the target collection from the field name (e.g. `organizationId` → `organizations`). Call `list_collections` to verify the target collection exists. Store:
```json
{ "foreignKey": { "targetCollection": "organizations", "confirmed": true/false } }
```

**Arrays:**
```json
[{ "$group": { "_id": null, "minLen": { "$min": { "$size": "$fieldName" } }, "maxLen": { "$max": { "$size": "$fieldName" } }, "avgLen": { "$avg": { "$size": "$fieldName" } } } }]
```
Store as `arrayDetails: { elementType, minLength, maxLength, avgLength }`. Determine element type from the sample data. If array elements are strings with <30 distinct values, unwind and count them as enum values too.

**Important:** You may batch related fields into a single `$group` stage where it makes sense (e.g. all number fields in one aggregation, all boolean fields in one aggregation) to reduce the number of MCP calls. Just make sure every field gets analyzed.

### Step 6: Change Frequency

If both `createdAt` and `updatedAt` fields exist in the collection, run:
```json
[
  { "$match": { "updatedAt": { "$exists": true }, "createdAt": { "$exists": true } } },
  { "$project": { "diff": { "$subtract": ["$updatedAt", "$createdAt"] } } },
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

If these fields don't exist, set `changeFrequency.hasTimestamps: false` and skip.

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
    "documentCount": <number>,
    "exploredAt": "<ISO 8601 timestamp>",
    "sampleSize": 100
  },
  "fields": {
    "<fieldName>": {
      "type": "<primary_type>",
      "presence": <0.0–1.0>,
      "description": null,
      ...type-specific keys from Step 5
    }
  },
  "aliases": {
    "<fieldName>": ["alias1", "alias2"]
  },
  "foreignKeys": [
    { "field": "<fieldName>", "targetCollection": "<collection>", "confirmed": true/false }
  ],
  "changeFrequency": {
    "hasTimestamps": true/false,
    ...metrics from Step 6
  },
  "notes": ["<user-provided notes>"]
}
```

Key formatting rules:
- Use **flat dot-notation** for nested fields (e.g. `address.city`, not nested objects)
- Mark sparse fields (<10% presence) with `"sparse": true`
- Mark mixed-type fields with `"mixedTypes": true` and list all observed types
- Set `"description": null` on every field as a placeholder for future annotation
- Mark user-confirmed deprecated fields with `"deprecated": true`
- All enum values should include their counts
- Foreign keys appear both on the field object and in the top-level `foreignKeys` array

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
