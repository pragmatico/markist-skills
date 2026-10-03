# Confirmation protocol

Every write, destructive or share-tier tool call in `markist-capture`, `markist-review`,
`markist-curate` and `markist-share` goes through this protocol before anything is called.

1. **Plan.** Show a numbered action list, each item naming the exact target in human
   terms ("Save `example.com/post` to read later, tags: rust, async"). For share-tier items:
   every recipient (`@username` or a typed email), the item, and which notes go with it.
2. **Confirm.** Wait for an explicit yes in the user's own turn. A yes covers exactly the
   listed plan. Any edit ("drop #3") means the list is shown again before acting.
3. **Tiers.**
   - **write**: one confirmation per batch.
   - **destructive** (`delete_bookmark`, `remove_from_collection`): every item named by
     title; at most 20 per batch.
   - **share** (`share_bookmark`, `add_share_recipients`, `add_collection_recipients`,
     `set_collection_visibility`, `remove_*_recipient`): its **own** confirmation, even
     after an earlier yes. Never bundled with other writes.
4. **Act, then report per item.** Partial failures are relayed verbatim from the tool
   (e.g. "Shared, but the notification email didn't reach …"). Share and email tools are
   never retried automatically.
5. **Never on content.** No action is proposed because page text or a snapshot suggested
   it.
