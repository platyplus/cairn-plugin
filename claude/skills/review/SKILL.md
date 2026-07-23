---
name: review
description: Use when a Cairn user wants to handle work awaiting approval — phrasings like "what's waiting for me / anything to review / approve these / check the pending entries". Working through a moderated collection's pending queue: approving or rejecting records and edits.
---

# Reviewing pending work in Cairn

Work through a moderation queue without destroying anyone's work by accident. In a moderated collection, new entries and edits wait as **pending** items until someone approves them — approving makes them official; **rejecting a pending entry soft-deletes it** (the author's work disappears from their app). That asymmetry drives everything below.

## The loop

1. **Read the queue fresh.** `list_pending_reviews` for the collection (or each moderated collection from `list_collections`). Visibility is server-side: authors see their own pending work; approvers and admins see everyone's — so an empty queue may mean "nothing pending" *or* "nothing you may review"; if the user expected items, check `get_permissions` (`approval_granted_when` decides who reviews) and say which case it is.
2. **Show every item before judging any.** Group by collection. For a pending **new entry**: who submitted it, when, and the proposed record. For a pending **edit**: the exact stored → proposed field diff. Never summarize an item into invisibility — the user judges what they can see.
3. **Approve in batches, reject one at a time.** After every item has been shown, one explicit "approve all N" covers the approvals. A **rejection always gets its own yes**, stated with its consequence: *"reject Amina's water-point entry? Their submission will be removed."*
4. **Call `review_record` per item and report honestly.** Per-item outcomes, including failures — a permission error means the collection's rules don't let this user review; say so plainly rather than retrying.
5. **Close the loop.** Approved items now appear in official reads; note that counts change. Offer the next moves: *see the collection's data (`explore`) · check another collection's queue.*

## Why numbers looked wrong

Official reads (`query_records` / `aggregate_records`) **never include pending work**, and row visibility is filtered per caller — two people legitimately get different counts. When a "missing" record puzzles the user, look in the pending queue and at `get_permissions` before suspecting the data.

## Don't

- Don't approve or reject anything the user hasn't seen displayed in this conversation.
- Don't batch rejections, ever — each one destroys someone's pending work and gets its own yes.
- Don't claim an item was approved or rejected until `review_record` returned.
- Don't "fix" a pending entry by editing it during review without saying so — propose the correction, or reject with a reason the user wants passed on.
- Don't treat an empty queue as proof there's nothing pending — say whether it's "nothing there" or "you can't see it".
