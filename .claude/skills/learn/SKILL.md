---
name: learn
description: Analyze daily query logs and extract learnings to enrich schema and guide files
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Learn

Analyze unprocessed daily query logs, extract explicit learnings and implicit patterns, and enrich schema and guide files so future query generation improves automatically.

**This skill takes no arguments.** It processes all unprocessed query log entries.

---

## Instructions

Follow steps 1–8 in order.

### Step 1: Read State

Read `memory/learn-state.json`. If the file doesn't exist, treat all query log files as fully unprocessed.

Also read `memory/guide.json` — you'll need it throughout.

### Step 2: Find Unprocessed Entries

Glob `memory/queries/*.json` to find all query log files. Compare each file against the state:

- **Not in state** → the entire file is new; all entries are unprocessed
- **In state but current array length > `entryCount`** → entries from index `entryCount` onward are new
- **In state and current array length == `entryCount`** → fully processed, skip

Read each file that has new entries and collect only the unprocessed entries.

If there are **zero** new entries across all files, print "No new queries to learn from." and stop.

Otherwise, print a one-line summary: "Found N unprocessed entries across M files."

### Step 3: Collect Explicit Learnings

Walk all unprocessed entries and collect `learnings` objects (if present). Group them by collection — use both the entry's `collection` field and any collections in `relatedCollections`.

Categorize each learning by its `type`:

| Learning type | Target |
|---|---|
| `field_discovery` / `field_usage` | Schema → field `description` (only if currently null) or `notes` |
| `business_rule` | Schema → `notes`; guide → `defaultFilters` if the detail contains "always filter" / "always exclude" / "always include" language |
| `correction` | Schema → `notes` |
| `enum_discovery` | Schema → field `enumValues` |
| `relationship` | Schema → `foreignKeys` (confirmed: true); guide → `references` |
| `gotcha` | Schema → `notes` |

### Step 4: Analyze Implicit Patterns

Across all unprocessed entries, derive:

1. **New relationships** — Look at `$lookup.from` values in each entry's pipeline, plus `relatedCollections`. For each collection, check if the relationship already exists in guide.json `references`. Collect any that are missing as new implicit relationships (these will be added with `confirmed: false`).

2. **Common techniques** — Count technique occurrences per collection (from the `techniques` field). Any technique appearing 2+ times for the same collection is a candidate for a schema note.

3. **Unexplored collections** — Collections referenced in `relatedCollections` or `$lookup.from` that don't have an entry in guide.json `collections`. These will get stub entries.

### Step 5: Present Summary

Show the user a structured summary of everything found:

```
## Learnings Summary

### Explicit Learnings
- <collection>: <count> learnings (<types>)

### Implicit Patterns
- New relationships: <list>
- Common techniques: <list>
- Unexplored collections: <list>

### Changes to Apply
- Schema files to update: <list>
- Guide.json changes: <list>
```

If there are no learnings or patterns of any kind, print "No actionable learnings found in the new entries." Update the state file (Step 8) and stop.

Otherwise, ask the user: **"Apply these changes?"** Wait for confirmation before proceeding to Step 6.

### Step 6: Enrich Schema Files

For each affected collection that has a schema file (check guide.json `schemaFile`):

1. Read the schema file
2. Apply changes:

**Field descriptions** (from `field_discovery` / `field_usage`):
- Only set if the field's current `description` is `null`
- Use the learning's `detail` as the description

**Notes** (from `business_rule`, `gotcha`, `correction`, common techniques):
- Before adding, check existing `notes` array for duplicate content (search for key phrases from the new note)
- Skip if a substantially similar note already exists
- Append new notes to the end of the array

**Enum values** (from `enum_discovery`):
- Merge with existing `enumValues` if present, don't overwrite

**Foreign keys** (from `relationship` learnings):
- Add to the `foreignKeys` array with `confirmed: true`
- Skip if the same field→targetCollection pair already exists

**Implicit foreign keys** (from Step 4 `$lookup` analysis):
- Add to the `foreignKeys` array with `confirmed: false`
- Skip if the same field→targetCollection pair already exists (whether confirmed or not)

3. Write the updated schema file back

### Step 7: Enrich Guide.json

Read `memory/guide.json` and apply:

1. **New references** — Add any new relationships (explicit with confirmed status, implicit from $lookup) to the appropriate collection's `references` map
2. **New defaultFilters** — From `business_rule` learnings that contain "always filter"/"always exclude"/"always include" language
3. **Stub entries for unexplored collections** — Add minimal entries:
   ```json
   {
     "aliases": [],
     "description": "",
     "documentCount": null,
     "defaultFilters": {},
     "deprecatedFields": [],
     "references": {},
     "schemaFile": null,
     "exploredAt": null
   }
   ```

Write the updated guide.json back.

### Step 8: Update State

Build the updated state object:

```json
{
  "lastRunAt": "<current ISO 8601 timestamp>",
  "processedFiles": {
    "<filename>": { "entryCount": <total entries in file>, "processedAt": "<current ISO 8601 timestamp>" }
  }
}
```

Include **all** query log files (not just newly processed ones) with their current entry counts.

Write to `memory/learn-state.json`.

Print a confirmation summary:

```
Learn complete:
- Processed: N new entries
- Schema files updated: <list or "none">
- Guide.json: <changes or "no changes">
- State saved to memory/learn-state.json
```
