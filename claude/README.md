# Cairn — Claude plugin

Drive [Cairn](https://cairn.platyplus.io) from Claude. Cairn is schema-driven, offline-first data for humanitarian and field teams: you describe what you want to track, Claude designs the collection, and your records sync and render as tables, charts, and maps — without leaving the conversation.

## Get started

1. **Install** (requires access to the private `plmercereau/cairn` repository):

   ```
   /plugin marketplace add plmercereau/cairn
   /plugin install cairn@platyplus
   ```

   > A public distribution channel (for users without repo access) is a planned follow-up.

2. **Start a conversation and just say hi.** Cairn greets you and walks you through a two-minute example, then helps you build the real thing you want to track. The first time Claude calls a Cairn tool you'll be asked to sign in once (run `/mcp` if prompted).

That's it — no tokens to copy, nothing to run locally.

## What's inside

One install bundles the Cairn connection plus six skills:

- **`getting-started`** — welcomes a new user and walks the full loop (build an example → add data → see it as a table, chart, and map), then pivots to your real data.
- **`design`** — turn "I want to track …" into built, checked collections — a single form or a whole project (photos of paper forms and spreadsheets welcome): agree on a plan, preview every form, then build and check it on your go.
- **`evolve`** — change what exists safely: add or rename fields (dry-run first, no silent data loss), adjust who sees what, edit skip-logic, or delete a record or collection with eyes open.
- **`ingest`** — load a batch of records safely: shape to the schema, dry-run, read the report, then insert.
- **`explore`** — answer questions about your data as the right table, aggregation, or chart (and hand off to a map for *where* questions).
- **`review`** — work through entries awaiting approval: see exactly what's pending, approve in batches, reject only with eyes open.

The connection is the remote Cairn MCP server (`https://cairn.platyplus.io/api/mcp`), wired up automatically.

## Try it

- "I want to track facility surveys — name, type, GPS location, and number of beds."
- "Import these rows into the health facilities collection." *(paste or point at a file)*
- "How many cholera cases by classification?" → a bar chart.
- "Which counties have the most cases?" → a choropleth map.
- "Add a follow-up date to the cases form." *(previewed, dry-run, then applied)*
- "Anything waiting for my review?"

## What you need

A Cairn account on the instance the connector points at (`cairn.platyplus.io` by default). Sign-in happens through Claude's sign-in flow on first use — no tokens to copy.

## Notes

- The skills are end-user *product* skills for driving Cairn; they are not development skills for this repository.
- The connection is a **remote** MCP server, so the plugin works the same on any machine — there is nothing to run locally.
