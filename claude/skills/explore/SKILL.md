---
name: explore
description: Use when a Cairn user asks a question about the data already in a collection — phrasings like "how many / which / show me / top N / trend over time / breakdown by ..." — answered as a table or an aggregation (counts, group-by, time trend).
---

# Exploring data in a Cairn collection

Answer a question about an existing collection with the right read: a **table** of rows or an **aggregation** (counts, group-by, trend). Reliability comes from getting the Cairn-extended CEL right for the query surface — not from guessing.

## The loop

1. **Read the schema + celContext first.** `get_data_schema` gives the fields, their kinds (which are `oneOf`/select, which are `x-references`, which are `x-calculation`/geometry), and the `celContext` catalog. Author against it.
2. **Classify the question → the right read.**
   - "show me / list / which rows" → **`query_records`** (a table).
   - "how many / count / top N / breakdown by / trend over time" → **`aggregate_records`** (grouped metrics; it also returns a chart-spec + `chartType`).
3. **Select fields explicitly — never omit `fields`.** A default projection can include a heavy geometry/blob or a calculated column and **fail the whole read**. Project only what you need; pull a reference's value with a **dotted path** (`residence.name`, `facility.name`).
4. **Write the filter in Cairn-extended CEL for the SQL surface** — see below.
5. **Show the result** plainly — the table, the grouped numbers, or the right **chart** (see *Choosing a chart*) — and **offer the next 2–3 moves** (filter/segment further · a different cut · map it for a *where* question).

## Choosing a chart

`aggregate_records` returns a `chartType` and the widget renders whatever you pass — including a poor choice. The one that reliably misleads: **a breakdown across categories** (by classification, by type, by area). Render it as **horizontal bars, largest-first — not a pie.** A pie makes the differences between slices hard to read, and gets worse with more categories. Keep bars honest: the value axis starts at zero.

## Rows you can('t) see

Reads are row-scoped server-side: a collection's `read_when` permission filters which rows exist *for this caller*, and `query_records`/`aggregate_records` return only **official** rows — pending (unapproved) work never appears in them. So two users can correctly get different counts from the same query — that's policy, not data loss. When a count or a missing row surprises the user, check `get_permissions` (or `get_guide('permissions')`) before suspecting the data; `list_pending_reviews` shows the pending work the caller is allowed to see.

## CEL on the query surface — select/oneOf and references

`query_records`/`aggregate_records` `filter` and `groupBy` compile to **SQL**.

- **Select/oneOf equality:** match by stored **code** with `dehydration.value == 'severe'`, or by human **label** with `dehydration.name == 'Severe'`. Both translate, and `validate_cel_expression` blesses them — pre-flight with these forms. (The bare `dehydration == 'severe'` also translates, but `validate` types the field as an object and flags it, so prefer `.value`/`.name`.)
- **Reference traversal is surface-specific:** dotted paths (`residence.name`, `facility.name`) project and group in `fields` and `groupBy` (group-by dotted paths are **reference-only** — a bare enum groups by its bare name), but a **reference** dotted path does **not** translate inside a `filter`. On read, a reference column already shows its label, so you rarely need `.name` in `fields`.
- Numeric/boolean/compound filters translate normally (`age > 60 && outcome.value == 'referred'`). For a **trend**, use `aggregate_records` `timeBin` (`{field, unit:"month"}`) — let SQL bin it; don't pull rows to bin by hand.
- **Date arithmetic translates too** — `visit_date.diffDays(today()) < 30`, `dob.addYears(18) <= today()` — so "in the last N days / older than / due by" questions are one filter, not client-side math. `get_guide('cel')` lists the full catalog with what the SQL surface supports; pre-flight anything unusual with `validate_cel_expression`.

## Don't

- Don't put a **reference** dotted path (`facility.name`) inside a `filter` — it doesn't translate; project/group it in `fields`/`groupBy` instead.
- Don't omit `fields` (a default projection can pull a heavy geometry/calculated column and fail the read).
- Don't pull all rows to count/bin what `aggregate_records` does in one call.
