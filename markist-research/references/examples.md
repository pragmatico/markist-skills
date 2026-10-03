# Worked examples

Four transcripts showing the shape of each retrieval mode and edge case. Tool calls are
abbreviated to their name and the arguments that matter.

## 1. Find mode

**User:** "What have I saved about Postgres full-text search?"

1. `search_bookmarks(query: "Postgres full-text search", limit: 30)`
2. `list_tags()` — no tag named anything close to "postgres" or "fts", so no `#tag`
   follow-up.
3. Three results come back; all clearly relevant on title/summary. This is a find-mode
   question ("what have I saved"), so no `read_bookmark_content` calls are made.

**Response:**

> Searched 1 way, matched 3.
>
> 1. **Postgres full-text search, the parts that actually matter** —
>    ts-rank-and-tsvector.example.com — matched on title and summary — in your library
> 2. **Why we moved off ILIKE for search** — infra-notes.example.com — matched on tags
>    (`postgres`, `search`) — read later
> 3. **GIN vs GiST for tsvector columns** — pg-internals.example.com — matched on summary
>    — in your library

## 2. Answer mode, with citations

**User:** "Based on my bookmarks, how should I tune pgvector for a small dataset?"

1. `search_bookmarks(query: "pgvector tuning small dataset", limit: 30)` → 4 candidates.
2. `list_tags()` → a `pgvector` tag exists, so a second search:
   `search_bookmarks(query: "#pgvector tuning")` → confirms the same 4 plus one more,
   merged and deduped to 5.
3. Answer mode (the user asked "how should I", not "what did I save") — reads the top 5,
   first chunk each: 5 `read_bookmark_content` calls, well under the 8-chunk cap.

**Response:**

> Searched 2 ways, read 5 of 5 matches.
>
> For a small dataset (under ~100k rows), skip HNSW and use a plain sequential scan or
> an IVFFlat index with a small `lists` value — the index overhead isn't worth it until
> the table is much bigger [1][2]. If you do index, keep `lists` close to
> `rows / 1000` and re-`ANALYZE` after bulk loads, since IVFFlat's planner estimate
> depends on it [3]. Your own note on [2] adds that you saw recall drop sharply below
> `lists: 10` on a 5k-row test table (your note).
>
> **Sources**
> [1] Why we skipped an index on our 12k-row vector table — pgvector-notes.example.com — https://pgvector-notes.example.com/skip-index
> [2] IVFFlat tuning, the parameters that matter — vector-tuning.example.com — https://vector-tuning.example.com/ivfflat (your note)
> [3] ANALYZE and why IVFFlat needs it — pg-internals.example.com — https://pg-internals.example.com/analyze-ivfflat

## 3. Thin coverage → offer

**User:** "What do my bookmarks say about WASM garbage collection proposals?"

1. `search_bookmarks(query: "WASM garbage collection proposal", limit: 30)` → 1 loosely
   related result (a general WASM intro that mentions GC in passing).
2. `list_tags()` → no matching tag.
3. Two reformulations (`"WebAssembly GC"`, `"wasm-gc"`) → no new results.

Fewer than 2 relevant bookmarks — this is thin coverage. No `read_bookmark_content`
calls; nothing here is actually about the topic, so there's nothing to cite yet.

**Response:**

> Searched 3 ways and found one loosely related bookmark (a general WASM intro that
> mentions garbage collection once, in passing) — not enough to answer this from your
> library. I can either (a) answer from general knowledge instead, clearly labelled as
> not coming from your bookmarks, or (b) look for good sources on WASM's GC proposal and
> offer to save them. Which would you like?

Only after the user picks (a) or (b) does the skill act — (a) writes a "Beyond your
bookmarks" answer with no citations; (b) hands off to finding/saving sources, itself a
write the user must confirm before anything is saved.

## 4. Two sources that disagree

**User:** "Per my bookmarks, is it safe to run migrations inside a transaction on
Postgres?"

1. `search_bookmarks(query: "Postgres migrations transaction safety", limit: 30)` → 3
   candidates.
2. Answer mode: reads all 3, first chunk each.

**Response:**

> Searched 1 way, read 3 of 3 matches.
>
> Your sources disagree on this. One says wrapping a migration in a transaction is
> always safe on Postgres because DDL is transactional there, unlike MySQL [1]. Another
> specifically calls out that `CREATE INDEX CONCURRENTLY` **cannot** run inside a
> transaction block at all, so a migration runner that wraps everything in one
> transaction will fail the moment it needs a concurrent index [2]. A third doesn't take
> a position but documents the same `CREATE INDEX CONCURRENTLY` restriction [3]. So: safe
> in general, except for concurrent index builds, which need to run outside the
> transaction.
>
> **Sources**
> [1] Postgres migrations are safer than you think — safe-migrations.example.com — https://safe-migrations.example.com/postgres-ddl
> [2] The migration runner bug that took down our index build — migration-bugs.example.com — https://migration-bugs.example.com/concurrent-index
> [3] Postgres docs notes: CREATE INDEX CONCURRENTLY — pg-internals.example.com — https://pg-internals.example.com/concurrently
