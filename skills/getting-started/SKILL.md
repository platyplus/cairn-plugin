---
name: getting-started
description: Use the first time a Cairn user arrives — phrasings like "hi / how do I start / get started / show me how Cairn works / what can this do", or any opening message when the user's workspace has no collections yet and they haven't named something specific to do. Welcomes a newcomer and walks them through a two-minute example end to end.
---

# Getting started with Cairn

Take a brand-new user from "just connected" to "I get it" in about two minutes, using a ready-made example — then pivot to the real thing they want to track. The win is a quick, complete loop (build → add data → see it), not teaching every feature.

## When to run this (and when not to)

1. **Check the workspace first.** Call `list_collections`. If it returns **any** collection, this user is not new — **do not** run the tour; help with what they asked, or hand to `design` / `ingest` / `explore`.
2. **Respect a stated goal.** If the opening message already names something to do ("I want to track water points", "import this CSV", "how many cases by district"), **skip the tour** and go straight to `design`, `ingest`, or `explore`.
3. Only run the welcome when the workspace is **empty** and the user is just saying hello or asking how to begin.

## The loop

1. **Greet and offer.** One warm line on what Cairn is — "you describe what you want to track, I build it, and your data shows up as tables, charts, and maps." Then offer: *"Want me to show you in two minutes with a quick example? I'll build a small one, then we'll make your real thing."* Wait for a yes.
2. **Build the example.** Read the bundled `facility-survey.example.json` (next to this skill). After a quick *"I'll create a small example collection called 'Example: Facility survey' — ok?"*, call `create_collection` with its `{ name, title, dataSchema }`. Creating requires project-admin; if the tool returns a permission error, say so plainly and offer to continue read-only or ask a project admin.
3. **Show the form.** Call `render_collection_form` with the new `collectionId` so they see the actual form they just made.
4. **Add data.** Add two sample facilities with `create_record` (point values are `{ lng, lat }`):
   - `{ name: "Central Hospital", facility_type: "hospital", beds: 120, location: { lng: 32.58, lat: 0.32 } }`
   - `{ name: "Riverside Clinic", facility_type: "clinic", beds: 12, location: { lng: 32.61, lat: 0.30 } }`
   Then invite them to add one of their own — *"give me a place name and I'll add it."*
5. **Show it come to life.** Lead with a **table** (`query_records`, select `fields` explicitly: `name`, `facility_type`, `beds`). Then a **bar chart** of count by type (`aggregate_records`, `groupBy: facility_type`). Then, as a flourish, the **points on a map** (`query_records` `view: 'map'`, `pointField: 'location'`). The table + chart are the win — if the map needs more setup, don't block on it.
6. **Pivot to their real thing.** Say *"that's the whole loop — now, what do you actually want to track?"* and hand into the `design` skill. (Once their real collection exists, `design` can also set who may see or edit its records and whether entries need review — worth one mention, not a demo.)
7. **Offer to tidy up.** Once their real collection exists, offer to remove the example — *"want me to delete the 'Example: Facility survey' collection?"*. Deleting needs the right permission; if they can't, the clear "Example:" label keeps it from being mistaken for real data.

## Don't

- Don't run the tour for a returning user (workspace has collections) or when the user already named a specific goal — defer to `design` / `ingest` / `explore`.
- Don't create the example, add records, or delete anything without a quick yes.
- Don't say the collection or a record exists until the tool returns.
- Don't stall the demo on the map — the table and chart already deliver the "aha".
