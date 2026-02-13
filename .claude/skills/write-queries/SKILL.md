---
name: write-queries
description: Log an executed MongoDB query with enriched metadata — stages, techniques, fields used, learnings, and tags
allowed-tools: Read, Write, Edit, Glob, Grep
---

# Write Queries

Log the most recently executed MongoDB query to the daily query log with enriched metadata for future pattern recognition and schema enrichment.

**This skill takes no arguments.** It works from conversation context — the query you just ran.

---

## Instructions

Follow steps 1–9 in order. Do not prompt the user — this is a silent logging operation.

### Step 1: Identify the Query

Look back through the conversation to find the most recently executed `run_aggregation` call. Extract:
- **collection** — the collection queried
- **pipeline** — the full aggregation pipeline
- **description** — write a plain-English summary of what the query does

If no query was executed in this conversation, print "No query found to log." and stop.

### Step 2: Read Guide

Read `memory/guide.json` to get `defaultFilters` for the collection. You'll need this in Step 6.

### Step 3: Extract Stages

Extract the top-level operator key from each pipeline stage (e.g. `$match`, `$group`, `$lookup`). Store as the `stages` array.

### Step 4: Describe Techniques

Write free-form descriptions of notable patterns used in the pipeline. No fixed vocabulary — describe what you see. Examples:
- "relative date filtering with $dateSubtract and $$NOW"
- "pipeline-form $lookup with let/pipeline"
- "conditional accumulation with $cond inside $sum"
- "timezone conversion via $dateToString"
- "$facet for parallel aggregations"

Skip this field if the pipeline is straightforward (simple match + project with no notable patterns).

### Step 5: Extract Fields Used

Walk the pipeline and record which fields appear in which stage type. Group by stage type:

- **match** — fields referenced in `$match` conditions
- **group** — fields used in `_id` and accumulators in `$group`
- **sort** — fields in `$sort`
- **project** — fields included/computed in `$project` or `$addFields`
- **lookup** — the `localField` or `let` fields in `$lookup`
- **unwind** — fields in `$unwind`

Only include stage types that appear in the pipeline. Use the field's short name (last segment), not the full path, unless ambiguous.

### Step 6: Check Default Filters Applied

Compare the `$match` stages against the `defaultFilters` from `memory/guide.json` for this collection.

List the field names from `defaultFilters` that appear in a `$match` stage. Store as `defaultFiltersApplied`.

If the collection has default filters but none were applied, still set `defaultFiltersApplied` to an empty array — this serves as an audit signal.

### Step 7: Auto-assign Tags

Assign one or more tags based on pipeline shape heuristics:

| Condition | Tag |
|-----------|-----|
| Has `$group` with `$sum`/`$avg`/`$count` | `analytics` |
| Has `$lookup` | `lookup` |
| Matches a specific `_id` or narrow filter | `debugging` |
| Has `$group` + `$sort` + `$limit` | `reporting` |
| Checks field existence, null, type | `data_quality` |
| Simple `$match` + `$project` only | `config_check` |

A query can have multiple tags. Pick whichever apply.

### Step 8: Collect Learnings

Review the conversation for any of these:
- **correction** — user corrected a field name, operator, or approach
- **field_usage** — discovered which field to use for a purpose (e.g. "use estimate_amount.total for revenue")
- **business_rule** — learned a domain rule (e.g. "always exclude VOID applications")
- **enum_discovery** — discovered valid enum values for a field
- **relationship** — discovered a link between collections
- **gotcha** — discovered a non-obvious behavior or caveat

Format each as `{ "type": "<type>", "detail": "<description>" }`.

Omit the `learnings` field entirely if there are none.

### Step 9: Write to Query Log

Assemble the entry:

```json
{
  "collection": "<collection>",
  "relatedCollections": ["<from $lookup.from fields>"],
  "description": "<plain english>",
  "stages": ["$match", "$group", ...],
  "techniques": ["<free-form descriptions>"],
  "fieldsUsed": {
    "match": ["field1", "field2"],
    "group": ["field3"]
  },
  "defaultFiltersApplied": ["field1"],
  "tags": ["analytics"],
  "learnings": [
    { "type": "correction", "detail": "..." }
  ],
  "pipeline": [...]
}
```

**Omit when empty** (do not write as `[]` or `{}`):
- `relatedCollections` — only include if pipeline has `$lookup`
- `techniques` — only include if there are notable patterns
- `learnings` — only include if there are learnings from conversation context

**Always include** (even if empty array):
- `stages`, `fieldsUsed`, `defaultFiltersApplied`, `tags`

Write to `memory/queries/YYYY-MM-DD.json` (today's date). If the file exists, read it and append to the array. If it doesn't exist, create a new array.

### Step 10: Confirm

Print a 2-line confirmation:

```
Logged: <description>
Tags: <tags> | Stages: <count> | Fields: <count> | Learnings: <count or "none">
```
