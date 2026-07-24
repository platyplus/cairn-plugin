---
name: design
description: Use when a Cairn user wants to start tracking something — one thing or a whole project. Phrasings like "I want to track / collect / register / make a form for X", or "set up my project / we need to track several things / here's our paper system". Covers a single new collection, a multi-collection setup with links and permissions, and adding collections to an existing project. Not for loading data (ingest), answering data questions (explore), or restructuring existing collections.
---

# Designing Cairn collections — one, or a whole project

Take "I want to track …" — one thing, or everything a project needs — to built, checked collections: previewed at every step, built only on an explicit yes. Draft fast and let the user react to visible things; never interrogate. Reserve questions for what you genuinely cannot infer: who sees what, whether entries need approval.

**Size the ask first.** One kind of thing → the **short path**: draft → preview → agree → build & check, four beats, no itinerary, no map, no breadcrumbs. Several kinds of things (or attachments implying them) → the **project path**. If intake changes the size, switch paths and say so. Ceremony is earned by scale, never applied by default.

<HARD-GATE>
Nothing that writes (`create_collection`, `commit_xlsform_import`, `set_permissions`, `update_data_schema`) is called until the plan has been shown in this conversation and the user explicitly said yes — for the project path, the complete re-rendered plan and one explicit go. Permission changes each need their own yes, at every size. No example record is ever saved — checks go through `dry_run_create_records` only.
</HARD-GATE>

## The short path (one collection)

1. **Draft a tight core.** Model only the fields the user named. Clear machine `name` (lowercase, ≤ 63 chars) + human `title`. Never add an `id` field; the uuid primary key is automatic.
2. **Preview before creating.** `render_collection_form` with `{ dataSchema, uiSchema }` — an *unsaved* draft, nothing is created. Iterate on the preview; offer extras *as questions the user accepts or declines* ("want a photo field? should the date be required?") — never bake them in unasked.
3. **Create only on an explicit yes.** `create_collection` with the agreed `{ name, title, dataSchema, uiSchema }`. Creating requires project-admin; a permission error is reported plainly.
4. **Check, then hand off.** Run the checks below, offer the permissions pass, then offer the next 2–3 moves: *fill in the form here · import a batch (`ingest`) · open it in the Cairn app.*

## The project path (several collections)

Open by showing the itinerary; keep a one-line breadcrumb on every message (e.g. *"Setup 3/5 — the case form"*). Every message ends with the step's refreshed visual and exactly one question. Off-path requests go to a visible **after-setup list** — acknowledge, park, return; hand them off at the end (`ingest` for imports, `explore` for questions, the Cairn app for the rest).

1. **Tell me about it.** The use case in their words, plus an invitation to show what they have — a photo of the paper form or register, a spreadsheet, an XLSForm (import it — see **Importing an XLSForm**). Read reality before planning: `list_collections`, and `get_data_schema` for anything the plan may link to.
2. **The map.** Propose the smallest set of collections and links that supports what people will actually enter next week. Before presenting, self-review: every collection, link, and field must trace to something the user said or attached — cut or park what doesn't. Columns extracted from documents get triaged: propose, ask, or park as legacy. Show the map as a small diagram (mermaid; existing collections grey, proposed ones highlighted — where diagrams don't render, an indented list). Everything cut or postponed goes on the after-setup list, visibly.
3. **Each form.** One collection at a time, using the short path's beats 1–2. After ~3 rounds on the same detail, offer to move on — *"we can refine this anytime later."*
4. **Who sees what.** Asked once for the whole project, in plain sentences (*"everyone on the project can see records; people edit only their own; new entries wait for approval"*), with per-collection exceptions only if the user raises them. Recommend the supervised model — new entries wait for approval — as the default; drop it without argument if declined. Offer **profile fields** here too — *"anything you want to track about your team members themselves (role, area, qualifications)?"* — a member-profile collection (`create_collection` with `kind: 'member_profile'`, at most one per project); its fields then power permissions via `_profile`.
5. **Build & check.** Re-render the complete plan (map + per-collection field lists + permission sentences) immediately before the gate — never gate on scrollback. On go: create collections one at a time in link order (referenced before referencing). Then permissions, one collection at a time, each with its own before → after confirmation. Then the checks, the honest report, and the after-setup hand-offs.

## Permissions (both paths)

Who may see/create/edit/delete records — and whether new entries need review — is six CEL predicates on the collection (`get_guide('permissions')` explains the model). Authoring is admin-only and gated:

1. `get_permissions` first — never propose a change blind; new collections already default `update_when`/`delete_when` to `created_by == _user` (authors touch only their own records), so say that instead of re-setting it.
2. Pre-flight each predicate with `validate_cel_expression` (target `permission`; target `read-floor` for `read_when`/`approval_granted_when`) before it appears in any plan.
3. Show the exact per-field **before → after** diff and call `set_permissions` only on its own explicit yes. Flag plainly when a change *widens* access (clearing `update_when`/`delete_when` lets any member touch any record) and that changing `read_when`/`approval_granted_when` makes affected members fully resync the collection.

## Checks (the green bar — every size)

For each created collection, synthesize a few realistic rows from the use case and run `dry_run_create_records` — including rows that must be REJECTED: a missing required field, an out-of-list value, a constraint violation. Green means "accepts what it should, refuses what it shouldn't, zero rows written." Show a failure as a failure; fix the schema or ask — never shrug it off. Checks across collections can run in parallel.

## Conditional fields (skip-logic)

"If X, also ask Y" (e.g. *if seen in a nest → habitat fields*) is authored **directly in the uiSchema** — add a `rule` to each dependent Control. No separate tool, no editor-mode switch needed at creation:

```json
{ "type": "Control", "scope": "#/properties/habitat_type",
  "rule": { "effect": "HIDE", "condition": { "expression": "in_nest != true" } } }
```

`render_collection_form` validates the rule. (Editing rules *later* in the app's simple Builder needs expert mode — but creating a collection that already has them does not.)

## Importing an XLSForm

An uploaded XLSForm *is* a collection draft — never retype it by hand.

1. **Draft.** `import_xlsform` (with the uploaded file's id) returns `{ dataSchema, uiSchema, summary }`, skip-logic already translated to CEL rules. `status: "blocked"` lists constructs Cairn doesn't support yet (with tracking issues) and yields no partial schema; `status: "invalid"` is a malformed workbook — report either plainly, don't paper over it.
2. **Preview and agree.** `render_collection_form` on the draft, then the gate as usual. A **cascading select** (choice_filter) also returns a `plan` of hierarchy-tree collections — say that the import will create those alongside the form.
3. **Commit on the yes.** With a `plan` → `commit_xlsform_import`, passing the draft's `dataSchema`, `uiSchema`, and `plan` back verbatim (it builds the tree collection(s) + the form's leveled reference). No `plan` → plain `create_collection`. Then run the checks.

## CEL is Cairn-extended, not standard

Rule conditions and `x-constraint`s use Cairn-extended CEL: reference field names directly; `select`/`oneOf` fields resolve to `{ value, name }`, so compare `.value` (e.g. `status.value == 'draft'`); geo/temporal helpers exist (`point.inside()`, `point.distanceTo(point)`). Before committing any non-trivial expression, read the collection's `celContext` and check it with `validate_cel_expression` — don't guess signatures. There is more on offer than equality checks — date windows, geo fences, tree relations, profile-gated rules, device-location capture (a Point field can default to the current position with `x-default: _location`), line geometry (a **LineString** field captures a drawn route or boundary; `line.length()` reads its length) — `get_guide('cel')` lists the full catalog with worked idioms; mention the fitting ones when the use case calls for them.

## Grounding

Every statement about the project's current state comes from a fresh read at that moment — on entry, before the gate, before executing, on resume. The conversation holds the plan; only tools know the project. Resuming (days later, or after a partial failure) means: re-read, reprint the itinerary with done/remaining, continue from what actually exists. There is no rollback — a partial build is reported as exactly that, with the resume point.

## Parallel or sequential

Reads, validations, previews, and dry-run checks: fan out in parallel freely. Writes: strictly one at a time, in link order. If it can change data, it waits its turn.

## Red flags (you are rationalizing)

| Thought | Reality |
| --- | --- |
| "They're in a hurry — I'll skip the preview this round." | The preview is the product. Never skip it. |
| "It's just one collection — the checks don't apply." | The green bar applies at every size. |
| "This register photo clearly implies six collections." | Propose the minimum; the rest goes on the after-setup list. |
| "They said 'looks good' about the map — that covers building." | The go gate is its own explicit question, always. |
| "One rejection check failed, but it's probably fine." | Red is red. Show it, then fix or ask. |
| "I remember what this project contains." | Memory holds the plan; `list_collections` holds the project. |
| "They asked about imports — I'll just do that quickly." | Park it on the after-setup list and return to the step. |

## Out of scope (say so, don't improvise)

- Saved views, tabs, dashboards, chart widgets — finished in the Cairn app; put them on the after-setup list.
- Changing or removing existing collections (fields, permissions, deletion) — that's `evolve`'s job; observations about existing collections go to the after-setup list.
- Loading data (`ingest`'s job), answering data questions (`explore`'s job), approving pending entries (`review`'s job).
- Creating or switching projects — the connection is bound to one project.

## Don't

- Don't pad the schema with fields, references, or constraints the user didn't ask for — **offer** them instead.
- Don't invent an `id` field.
- Don't ask a second question before the first is answered, or introduce a new topic mid-step.
- Don't show raw permission expressions when sentences will do — but always validate the real expressions behind them.
- Don't say anything was created until the tool returned — and read back what exists before reporting done.
