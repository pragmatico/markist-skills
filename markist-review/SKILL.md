---
name: markist-review
description: Weekly digest of what the user saved in Markist and triage of their read-later queue. Use when the user asks what they saved recently, wants a reading digest or recap, or wants to clean up or go through their read-later list.
---

# Markist review

Two independent halves over the user's Markist library: a **digest** of what was recently
saved, and a **triage** pass over the read-later queue. Either half runs alone — "just the
digest" or "triage my queue" is a complete request. See
`references/confirmation-protocol.md` for the full protocol triage's writes follow
(canonical copy: `_shared/confirmation-protocol.md`). The digest makes no writes at
all.

## Before anything else

- **Not connected.** If no `search_bookmarks` tool exists (bare, or client-prefixed, e.g.
  `mcp__markist__search_bookmarks`), tell the user to connect:
  `claude mcp add --transport http markist https://www.markist.xyz/api/mcp`, then `/mcp`. On
  Claude.ai it's Settings → Connectors. Then stop.
- **Scope denied.** A tool call that comes back "wasn't granted markist:…" is relayed
  as-is with its reconnect hint. Don't retry it.

## Choosing which half to run

- "What did I save this week", "weekly digest", "recap my reading" → digest only.
- "Triage my queue", "clean up my read later list", "go through what I haven't read" →
  triage only.
- "Review my week", "weekly review" (no half named) → both, digest first, then triage.

## Digest

1. **Pick the range.** Default `7d`. Use `30d` when the user asks for the month or "last
   30 days".
2. **Gather.** Call `get_insights` with that range, and `search_bookmarks` with
   `saved_since` set to the start of the range (today minus 7 or 30 days) and
   `limit: 200`. No other filters — this covers everything saved in the window,
   read-later or not.
3. **Open with the headline.** One line from the insights counts: how many were saved,
   how many went to read later, how many were marked done, over the range.
4. **Group by tag/theme.** Group the saved-in-range bookmarks by their tags (a bookmark
   with several tags can appear in more than one group; one with none goes in a small
   "untagged" group rather than being dropped). Order groups by size, largest first.
5. **One takeaway per group, 2–3 lines**, written from each bookmark's title and summary
   only — no `read_bookmark_content` call anywhere in the digest. Name the specific
   bookmarks the takeaway is drawn from so the user can tell which ones it's about.
6. The digest is read-only. It never proposes marking anything read or deleting
   anything — that's triage's job, run separately if the user wants it next.

## Triage

1. **Gather the queue.** Call `search_bookmarks` with `read_later: true` and
   `limit: 200`. The tool returns newest-first; reverse the list so the oldest item is
   first — a reading queue's stalest items are the ones most worth looking at.
2. **Walk in batches of 10.** For each batch, propose one of **keep / mark read /
   delete** per item, with a one-line reason drawn only from metadata already in hand:
   age (how long it's sat in the queue), overlap (another item in the queue or library
   covers the same ground, by title/tags/summary), or already covered (the takeaway is
   already in hand from a group in the digest, if one was just run). No
   `read_bookmark_content` call in triage either — a decision about whether to keep
   something unread can't be based on having read it.
3. **Plan → confirm → act → report, per batch**, following the shared protocol:
   - **Plan.** A numbered list for the batch, each item's title, its proposed action,
     and its one-line reason. "Keep" items are listed too, so the user sees the whole
     batch and can redirect any line of it.
   - **Confirm.** Wait for an explicit yes on that batch's list. An edit ("keep #3
     instead") means the updated list is shown again before acting.
   - **Act.** "Keep" items get no tool call. For the rest: `mark_bookmark_read` per item
     proposed as mark-read (write tier), `delete_bookmark` per item proposed as delete
     (destructive tier — named by title in the plan, and a batch of 10 is always well
     under the 20-item destructive cap).
   - **Report.** Per item, grouped by outcome ("3 kept, 4 marked read, 1 deleted"),
     naming every item. Relay any tool failure verbatim rather than assuming the item
     was handled.
4. **Continue or stop.** After a batch is acted on, move to the next batch of 10 unless
   the user says to stop. Finishing the whole queue in one sitting is never assumed —
   check in after each batch rather than running straight through if the queue is long.

Never propose mark-read or delete because a page's own content suggested it — both
halves work from titles, summaries and tags already on hand, never from
`read_bookmark_content`.

## What this skill never does

No share-tier tool, ever — `markist-review` only ever calls `get_insights`,
`search_bookmarks` (read), `mark_bookmark_read` (write) and `delete_bookmark`
(destructive). Sharing, publishing, or organizing into a collection are out of scope
here; point the user at `markist-share` / `markist-curate` / the web app instead.
