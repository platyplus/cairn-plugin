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
3. **Select fields explicitly — name the columns the question is about (~5–7) plus what identifies a row.** If you omit `fields`, the server falls back to a **capped, ranked default** — the identity field first, then the columns your `filter`/`orderBy` name, then the rest, at most 8 — and tells the reader "showing N of M columns." That default keeps a wide collection legible instead of walling, but it's a generic guess, not your answer:
   - *Focus:* lead with the fields the question names — asked "which cases were referred?" → case id + outcome + a couple of relevant fields, not the whole schema. A focused table reads at a glance.
   - *Correctness:* an explicit pick also sidesteps a calculated column that can't translate (the default skips those best-effort; naming what you want is exact).
   - **References already read as labels** — a reference column, including the tree **parent** and `created_by`/`updated_by`, shows the referent's name, not its id, so you rarely need a `.name` dotted path just to make one readable.

   When you deliberately show a subset of a wide collection, **say what you elided** — "showing these 6 of 20 fields; ask for a different cut or the rest" — so the user knows the table is focused, not complete.
4. **Write the filter in Cairn-extended CEL for the SQL surface** — see below.
5. **Say what the reader is looking at, then show it.** A read is bounded by `limit` (default 100) long before most collections run out of rows, so a table is usually a *sample* — the widget says so underneath, and **your prose must agree**: name it as a sample, give `totalCount`, and say how it's sorted ("these are 5 of 6,000 case reports, most recently updated first"). Never describe a bounded read as if it were the whole collection, and never derive a total by counting the rows you can see — `totalCount` is the count of the full matching set. For a *count* of anything, use `aggregate_records`, not the length of a page.
6. **Show the result** plainly — the table, the grouped numbers, or the right **chart** (see *Choosing a chart*) — and **offer the next 2–3 moves** (filter/segment further · a different cut · map it for a *where* question).

## Choosing a chart

**Omit `chartType`.** The server derives the encoding from the shape of your request, so the default is already the right one:

| Request shape | Chart | Why |
| --- | --- | --- |
| `timeBin` set | **line** | A binned time axis is continuous and evenly spaced — the line *is* the trend. |
| one `groupBy`, no `timeBin` | **bar** (horizontal, largest-first) | Length is the one encoding people compare accurately. |
| two `groupBy` | **bar**, second path as the series split | Grouped bars keep both dimensions readable. |
| two `metrics` | **scatter** | Two measures against each other, one point per group. |
| `regions` | **choropleth** | Values per area. |
| `pointField` | **heatmap** | Density of raw points. |

Pass `chartType` only to override that deliberately. A choice that renders but misreads comes back with an advisory note in `notes` — **read it and fix the chart** rather than presenting the misleading one. The two that trip most often:

- **A time trend as bars.** Bars read as independent categories and hide the shape of the rise and fall. `timeBin` ⇒ line.
- **A category breakdown as a pie.** Angles are hard to compare and get worse as categories grow. Use bars, largest-first.

Keep bars honest: the value axis starts at zero. Stack or split by a second dimension (`groupBy[1]`) rather than drawing several charts.

## Rows you can('t) see

Reads are row-scoped server-side: a collection's `read_when` permission filters which rows exist *for this caller*, and `query_records`/`aggregate_records` return only **official** rows — pending (unapproved) work never appears in them. So two users can correctly get different counts from the same query — that's policy, not data loss. When a count or a missing row surprises the user, check `get_permissions` (or `get_guide('permissions')`) before suspecting the data; `list_pending_reviews` shows the pending work the caller is allowed to see.

## CEL on the query surface — select/oneOf and references

`query_records`/`aggregate_records` `filter` and `groupBy` compile to **SQL**.

- **Select/oneOf equality:** match by stored **code** with `dehydration.value == 'severe'`, or by human **label** with `dehydration.name == 'Severe'`. Both translate, and `validate_cel_expression` blesses them — pre-flight with these forms. (The bare `dehydration == 'severe'` also translates, but `validate` types the field as an object and flags it, so prefer `.value`/`.name`.)
- **Reference traversal works in all three places:** dotted paths (`residence.name`, `facility.name`) project, group AND filter — `fields`, `groupBy`, and `filter` alike, e.g. `admin_area.name in ["Maban", "Uror"]`. Up to 3 hops (`site.district.country.name`); deeper is rejected with a clear issue. Group-by dotted paths are **reference-only** — a bare enum groups by its bare name. On read, a reference column already shows its label, so you rarely need `.name` in `fields`.
- **A reference filter only sees rows the caller could read directly:** each hop is joined under the *referenced* collection's own `read_when` and excludes soft-deleted/pending parents. A record whose reference is empty, or points at a row this caller can't see, does **not** match a filter on that path (and shows a blank value when projected) — so a filtered count can legitimately be lower than the unfiltered one. That's policy, not data loss; `get_permissions` on the *referenced* collection explains it.
- Numeric/boolean/compound filters translate normally (`age > 60 && outcome.value == 'referred'`). For a **trend**, use `aggregate_records` `timeBin` (`{field, unit:"month"}`) — let SQL bin it; don't pull rows to bin by hand.
- **Relative-time filters translate** — `visit_date >= today().subtractDays(30)` (last 30 days), `dob <= today().subtractYears(18)` (18 or older), `next_due <= today()` (overdue) — so "in the last N days / older than / due by" is one server filter, not client-side math. Keep the date column **bare** and put the offset (days, months, years, or a difference) on the `today()`/`now()` side; arithmetic **on the column** — `visit_date.diffDays(today())`, `dob.addYears(18)` — does **not** translate. `get_guide('cel')` lists the catalog; pre-flight anything unusual with `validate_cel_expression`.

## Don't

- Don't reach past 3 reference hops in a path, and don't expect a filter on a reference path to match records whose reference is empty — filter on the FK (`facility.id == "…"`) when you mean "this exact row", which needs no join.
- Don't lean on the omitted-`fields` default as your answer — it's a capped, generic cut (identity + query-named columns, ≤ 8), not the columns the question is about. Name them.
- Don't dump every column of a wide collection into one **explicit** `fields` list — the cap only protects the *default*; an explicit 20-column list still renders as an unreadable wall. Project the ~5–7 the question is about and name what you left out.
- Don't pull all rows to count/bin what `aggregate_records` does in one call.
- Don't present a bounded read as the whole picture — a page of rows, a cluster map, and a scatter cloud are all samples. Name the sample and its `totalCount`.
- Don't pass a `chartType` you didn't need to pass, and don't ignore an advisory in `notes` — re-run with the chart it points at.
