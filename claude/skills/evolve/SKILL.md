---
name: evolve
description: Use when a Cairn user wants to change or remove something that already exists — phrasings like "add a field / rename a column / change the type / make it required or unique / change who can see or edit records / add skip-logic to the form / delete this record / delete the collection". For creating new collections, use design instead.
---

# Evolving a Cairn collection

Change a live collection — its fields, its rules, who sees what, or its existence — without losing data by accident. These are changes to real tables holding real records: **dropped columns lose their data permanently, a mishandled rename is a silent drop, and there is no undo.** Slow is smooth.

<HARD-GATE>
No write (`update_data_schema`, `set_permissions`, `set_rule`/`remove_rule`, `set_editor_mode`, `delete_record`, `delete_collection`) until its preview — the dry-run plan, the before → after diff, or the exact thing being deleted — was shown in this conversation and the user explicitly said yes to it.
</HARD-GATE>

## Ground first

Start every change from a fresh read — `get_data_schema` (plus `get_permissions` when touching who-sees-what), never from memory or scrollback. If the conversation paused, re-read before executing: someone may have changed the collection meanwhile.

## Changing the fields (schema)

1. Build the **complete** new schema starting from `get_data_schema` — a property omitted from it becomes a `DROP COLUMN`.
2. **Always** `dry_run_update_data_schema` first, and read the plan back in plain words: what's added, what's changed, **`droppedColumns`** (their stored data is destroyed — name each one), and **`inferredRenames`** — confirm every rename with the user and **resend it explicitly via `renames`**. An ambiguous rename left unhinted silently degrades to drop-and-recreate: the column's data is gone.
3. `blockers` are changes the server refuses (lossy type change, making an existing column required): explain them, don't reshape the schema to dodge them silently.
4. A uniqueness rule whose dry-run reports conflicts will save but **not be enforced** — tell the user which existing records collide.
5. Only after an explicit yes on that exact plan: `update_data_schema` with the **same** schema and renames. Report what the tool returned, including any `unenforcedRules`.

## Changing who sees what (permissions)

Same discipline as at creation (the `design` skill): `get_permissions` first, speak in plain sentences (*"everyone can see records; people edit only their own"*), `validate_cel_expression` every predicate before it appears in a proposal, then show the exact per-field **before → after** diff and call `set_permissions` only on its own yes. Flag plainly when a change **widens** access (clearing `update_when`/`delete_when` lets any member touch any record) and that changing `read_when`/`approval_granted_when` makes affected members fully resync the collection.

## Changing the form (rules & skip-logic)

`set_rule` / `remove_rule` edit conditional show/hide on the form; preview with `render_collection_form` after each change. Switch `set_editor_mode` only after the user agrees — never silently.

## Removing things

- **A record:** show the exact record first (`query_records`), get a yes for that record, then `delete_record`. If the server refuses because other records reference it, report that reason — the referencing records must change first; don't retry.
- **A collection:** admin-only and the most destructive act in Cairn — the table and **every record in it** are destroyed, no undo. State the collection name and its record count (`aggregate_records`), get an explicit yes that **names the collection**, never folded into a broader confirmation. If deletion is blocked because another collection references it, say so and offer to remove the referencing field first (a schema change, above).

## CEL can do more than equality

Constraints, defaults, calculations, rules, and permissions all take Cairn-extended CEL — profile-gated access (`_profile.is_supervisor == true`), geo fences (`location.inside(zone)`), date windows (`visit_date.diffDays(today()) < 30`), tree relations (`isDescendantOf(area, region)`). `get_guide('cel')` lists the full catalog; read the collection's `celContext` and `validate_cel_expression` before proposing any of it.

## Don't

- Don't send `update_data_schema` without a dry-run of the **same** payload shown and approved in this conversation.
- Don't let an inferred rename pass unconfirmed — resend confirmed renames explicitly.
- Don't touch permissions without the before → after diff and its own yes.
- Don't delete anything the user hasn't seen named (and, for collections, counted) in this conversation.
- Don't claim a change applied until the tool returned — and read back reality before reporting done.
- Don't create new collections here — that's `design`'s job.
