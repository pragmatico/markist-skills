---
name: markist-curate
description: Build a Markist collection on a topic from the user's bookmarks, with curator notes. Use when the user wants to curate, assemble or organize their bookmarks on a topic into a collection or reading list.
---

# Markist curate

Builds a Markist collection — a named, ordered set of bookmarks with optional curator
notes — from the user's own library, on a topic they name. See
`references/confirmation-protocol.md` for the full protocol this skill follows
(canonical copy: `_shared/confirmation-protocol.md`).

## Before anything else

- **Not connected.** If no `search_bookmarks` tool exists (bare, or client-prefixed, e.g.
  `mcp__markist__search_bookmarks`), tell the user to connect:
  `claude mcp add --transport http markist https://www.markist.xyz/api/mcp`, then `/mcp`. On
  Claude.ai it's Settings → Connectors. Then stop.
- **Scope denied.** A tool call that comes back "wasn't granted markist:…" is relayed
  as-is with its reconnect hint. Don't retry it.

## Build: find the candidates

Run the same retrieval loop `markist-research` uses, in **find mode only**:

1. Run `search_bookmarks` with the user's phrasing. Call `list_tags` once; if a tag name
   matches the topic, also search with that `#tag`. Add up to 2 reformulations (a
   synonym, a narrower term) if results look thin. **At most 4 searches, `limit: 30`
   each.**
2. Dedupe the results by id, then rank by relevance to the topic using title, summary,
   tags and notes already on hand — `search_bookmarks` already fuses semantic and keyword
   ranking, so no extra re-ranking is needed.
3. **No `read_bookmark_content` call** unless the user explicitly asks for sharper
   reasoning on why a candidate belongs (e.g. "look closer before you decide"). Even then,
   a content read only sharpens the internal "why it's in" line below — it never feeds a
   curator note (see "Curator notes" below, which is a hard rule with its own source
   limits).
4. Propose **5–15 entries, defaulting to about 10**, and never more than the collection's
   50-entry cap. Each candidate gets a one-line "why it's in" drawn from its title, summary
   or matching tag.

Fewer than 2 relevant bookmarks → say so plainly rather than forcing a thin collection;
offer to broaden the topic or stop.

## Target: new or existing collection

Ask which, if it isn't already clear from the request.

- **New collection.** Draft a name (≤ 80 characters), a description (≤ 500 characters)
  and up to 10 tags from the topic and the candidate set. These are drafts shown in the
  plan for editing, not sent to any tool until confirmed.
- **Existing collection.** Call `list_my_collections`, match it to what the user named
  (ask if ambiguous), then `get_collection` for its current entries. Drop any candidate
  already in it (matched by url) before building the plan — it's already there.

## Order: foundational → advanced

Order the proposed entries foundational → advanced by default, or however the user asks
(e.g. "newest first"). Get this right **before** anything is added: entries land at the
end of the collection in the order they're inserted, and reordering afterwards means one
`move_collection_item` call per one-step swap — expensive compared to inserting in the
right order the first time. Adding to an existing collection appends the new entries
after whatever is already there; the two groups aren't interleaved.

## Curator notes are public — never from private notes

**Hard rule.** A curator note is text anyone with access to the collection can read. Each
drafted note:

- comes **only** from that bookmark's title and summary — never from the user's private
  bookmark notes, and never from page content even if a content read happened above;
- is ≤ 500 characters;
- is shown as an editable draft in the plan, not written until the user confirms it
  (plain style, not a sales pitch — a sentence on what the piece covers and why it's
  useful in this collection's context).

A candidate with nothing worth saying beyond its own title gets no curator note rather
than a padded one — an empty note is a valid outcome of `set_curator_note`.

## Plan → confirm → act → report

This is a single **write**-tier action (one confirmation per batch, per the shared
protocol):

1. **Plan.** Show: the target (new collection's name/description/tags, or the existing
   collection's name), the numbered entry list in final order with each one-line
   "why it's in", and each entry's draft curator note (or "no note" where none is
   proposed).
2. **Confirm.** Wait for an explicit yes. Any edit ("drop #4", "reword the note on #2",
   "move #5 before #2") means the updated plan is shown again before acting.
3. **Act, in order:**
   - New target: `create_collection` (always created **private** — publishing is a
     separate ask, below).
   - `add_to_collection` once per confirmed entry, in the confirmed final order.
   - `get_collection` once to read back the entries' ids (matched by url), then
     `set_curator_note` once per entry that has a confirmed note.
4. **Report.** Link `https://markist.xyz/collections/<slug>/edit` — the owner-only
   management page — and summarize what landed: the collection name, how many entries
   were added, and how many already-present ones were skipped (existing-collection case).
   Relay any tool failure verbatim rather than assuming an entry was added.

## Publishing and recipients: their own ask, their own confirmation

Never bundled with the build confirmation above, even right after a yes. Only on an
explicit, separate request:

- **Publish** ("make it public"): `set_collection_visibility` with `public`. Explain that
  anyone with the link can then view it.
- **Invite specific people** (private collection stays private, named people get access):
  `add_collection_recipients`. Restating `markist-share`'s recipient rules here, since a
  skill can't assume another one is installed:
  - Call `list_contacts` and match the user's words ("Sam", "the design team") to
    contacts. An ambiguous match is asked about, never guessed.
  - Each recipient is passed as `@username` or a contact id; a raw email is only used
    when the user typed one themselves.
  - The plan shows `@username` for a username contact and never its resolved email.
  - Mention the 20-recipient cap and the 50-emails-per-hour rate limit when the batch is
    large enough that either could matter.
- Each of these gets its **own** plan → confirm → act → report pass, share tier, per the
  shared protocol — never folded into the entry-adding confirmation.

## What this skill never does

No destructive tool, ever. `markist-curate` only ever calls `search_bookmarks`,
`list_tags`, `read_bookmark_content` (only on explicit request, never for curator-note
text), `list_my_collections`, `get_collection`, `create_collection`, `add_to_collection`,
`set_curator_note`, `set_collection_visibility` and `add_collection_recipients` — the last
two only on their own explicit ask. Removing entries, deleting a collection, or revoking a
recipient are out of scope here; point the user at the web app instead.
