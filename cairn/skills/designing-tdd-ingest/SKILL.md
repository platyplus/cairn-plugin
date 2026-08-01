---
name: designing-tdd-ingest
description: Use when a Cairn user wants to load data they need to *trust* — a recurring or pipeline-fed import where correctness must be checked, not assumed. Phrasings like "make sure the import is right / the totals should match / verify these loaded correctly / this runs every week / import from my pipeline and check it / did anything get dropped". Adds an assertion-first discipline — state the expected counts, coverage, categories, and shape *before* loading, then dry-run, verify, land, and re-verify — on top of a plain import. For a one-off paste-and-go load with no correctness to prove, use `ingest` instead.
---

# Loading data under test — assertion-first ingestion

State what a good load looks like *before* you touch the data, then let the tools prove it. The win is **trust in a repeatable load**, not speed: a pipeline breaks in slow, quiet ways — a column gets renamed upstream, a code list is truncated, coordinates flip lat/long, row counts drift, a re-run double-inserts. The remedy is TDD's: write the test first, then make it green.

This skill owns *assert → dry-run → verify → land → re-verify*. It does **not** own getting the rows (that's your own pipeline/tool stack) or the mechanics of shaping rows to the schema (that's the `ingest` skill — lean on it, don't re-teach it).

## The loop

1. **Agree the assertions first — red before green.** Before shaping or loading anything, propose a small set of *expected-shape* assertions and get an explicit yes. Use **soft thresholds, not strict equality** — real-world data is noisy. Five kinds cover most loads:
   - **row-count** — "between 800 and 1200 rows land."
   - **coverage** — "≥ 95% of rows have a GPS location", "every row has a `case_id`."
   - **taxonomy** — "`classification` is only ever `none` / `some` / `severe`" (no new or misspelled categories).
   - **geo-shape** — "every point falls inside the country's bounding box" (catches flipped lat/long).
   - **idempotency** — "re-running this load adds **zero** net-new rows."
2. **Shape rows to the schema — the `ingest` skill's job.** `get_data_schema`, exact field names, a select value is its `const`, **omit `x-calculation`**, resolve `x-references` by lookup, rows are `{ id, data }`. The one addition here: **set a deterministic `id` per row, derived from the row's natural key** (e.g. a hash of `case_id`) so the load is idempotent and a re-run is detectable.
3. **Dry-run, then check the assertions you can from the report.** `dry_run_create_records` writes nothing and returns counts plus per-row `validation_error` / `duplicate`. Check **row-count**, per-row validity, and dry-run **duplicates** against your assertions now. On a failed assertion or an ambiguous value (e.g. a `moderate` where the schema allows only `none`/`some`/`severe`), **stop and surface it** — propose a fix or escalate. Never coerce or drop a row to make an assertion pass.
4. **Land the batch — only on an explicit yes.** `bulk_create_records` with the *same rows and ids* you dry-ran. There is no separate "batch" tool: a **moderated** collection routes new rows to its review queue automatically via `create_when` — that *is* the moderation path.
5. **Re-verify against what actually landed — the part a plain import skips.** A green dry-run is not proof the rows are right in the table. Prove each assertion on live data with the read tools (this is the `explore` surface):
   - **row-count / coverage / taxonomy** → `aggregate_records` (group-by + counts). Render a category breakdown as **horizontal bars, largest-first — not a pie**.
   - **geo-shape** → `query_records` projecting the point field with explicit `fields`; check the bounds.
   - **idempotency** → `query_records` the **business key** (e.g. `case_id`) and look for collisions.
   - **Moderated collection?** The rows you just landed are **pending** and will **not** appear in `query_records` / `aggregate_records` until approved — reads return only official rows. Don't read an empty result as a failed load: re-verify after review, or check `list_pending_reviews`.
   Report each assertion **green/red with its number**. A red assertion after landing is a *finding*, not a formatting choice — say so plainly.
6. **Hand off.** Offer the next 2–3 moves: *re-run the load (it's idempotent) · explore the data · load the next source.*

## Idempotency & the re-run trap

A "run" is one batch you can re-execute safely; the idempotency assertion is that a second run lands **zero** net-new rows. The trap that breaks it:

- `dry_run_create_records`'s `duplicate` status keys on the row **`id`**, **not** on any `x-unique` business key. So a stable, deterministic `id` per row (step 2) is what makes a re-run report priors as `duplicate` instead of double-inserting.
- A clean dry-run does **not** prove there's no business-key collision. A row with the same `case_id` but a fresh `id` reads as `ok` in the dry-run and collides only at insert — so when re-loading against existing data, `query_records` the business key **first**.

## Don't

- Don't load before the assertions are agreed — stating expectations first *is* the discipline.
- Don't call a load "done" or "green" on the dry-run report alone — re-verify against live (or pending) data.
- Don't coerce an invalid value or silently drop a row to make an assertion pass — surface it and ask.
- Don't reach for a `create_batch` / lineage / `register_ingest_source` tool — none is exposed; land with `bulk_create_records`, moderate with `create_when`.
- Don't encode your pipeline tool's specifics here — sourcing and transforming rows is your own stack; this skill owns only *assert → verify → land → re-verify*.
