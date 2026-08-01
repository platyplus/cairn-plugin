---
name: ingest
description: Use when a Cairn user wants to load a batch of existing records into a collection that already exists — phrasings like "import these / load this data / add these rows / bulk upload" (rows pasted, generated, or from a file).
---

# Importing data into a Cairn collection

Get a batch of rows into an existing collection without corrupting it. The win is reliability, not speed: shape to the schema, dry-run, read the report, and never write blind. (Sourcing or generating the rows is your own research — this skill owns *shape-to-schema + safe insert*.)

## The loop

1. **Fetch the schema.** `get_data_schema` for the collection. Note the required fields, the `oneOf`/enum values, any `x-unique` business key, `x-references`, and `x-calculation` fields.
2. **Shape rows to the schema.** Use exact field names; a `oneOf`/select value is its `const` (e.g. `dehydration: "severe"`). **Omit every `x-calculation` field** — the server computes those. A row is `{ id, data }`: put the data columns in `data`; never put an `id` *inside* `data`.
3. **Resolve references by lookup.** A reference field (`x-references`) stores the target row's **uuid**, not its name. To map a name → id, `query_records` the referenced collection projecting its label field (e.g. `fields: ["name"]`) — the row `id` is returned **automatically**. **Don't project `id`** (it's reserved and errors). Put the looked-up uuid in the field; never leave a named link blank or invent an id.
4. **Dry-run, then read the report.** Call `dry_run_create_records` (writes nothing). Show the user the counts and every per-row `validation_error`/`duplicate`. On a `validation_error` or an ambiguous value (e.g. `dehydration: "moderate"` when the schema allows none/some/severe), **ask — don't silently coerce or drop the row.** A row can also fail as *"You are not allowed to create this record"* — that's the collection's `create_when` permission (admins bypass), not a data problem: report it, don't reshape the row to dodge it (`get_permissions` shows the rule).
5. **Insert only on an explicit yes.** `bulk_create_records` with the *same rows and ids* you dry-ran. Then **verify**: report inserted vs. failed counts honestly; never claim "done" without checking.
6. **Hand off.** Offer the next 2–3 moves: *explore the data · import another batch · open the collection in the app.*

## Idempotency & duplicates — the trap

The dry-run's `duplicate` status keys on the row **`id`**, **not** on any `x-unique` business key:

- **Set a stable, deterministic `id` per row** (derive it from the row's natural key). Re-running then reports prior rows as `duplicate` instead of inserting them twice.
- A clean dry-run does **not** prove there's no business-key collision. For a re-import keyed on a unique field (e.g. `case_id`), **`query_records` for those keys first** — a same-key/new-id row reads as `ok` in the dry-run but collides at insert.

## Don't

- Don't insert without a dry-run, or claim success without verifying counts.
- Don't hand-retype a uuid — use ids exactly as the tools return them.
- Don't coerce an invalid value or drop a row silently — surface it and ask.
- Don't write `x-calculation` fields, or an `id` inside `data`.
