# Markist agent skills

Five [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
that turn the Markist MCP server into workflows: research, capture, review, curate and
share. Each is a `SKILL.md` plus optional `references/`, no scripts — the logic stays in
the MCP tools and the service layer behind them; a skill only decides which tools to call,
in what order, and what the user sees.

## Prerequisite: the Markist MCP server

Every skill here is a workflow over Markist's MCP tools (`search_bookmarks`,
`add_bookmark`, `share_bookmark`, …). Without a connection, each skill explains how to
connect and stops — none of them fall back to the CLI, the web app, or a general web
search, since an answer has to come from *your* bookmarks.

Connect once, then every skill below can use it:

- **Claude Code**: `claude mcp add --transport http markist https://markist.xyz/api/mcp`, then `/mcp` to authorize.
- **Claude.ai / Desktop**: Settings → Connectors → Add connector → paste `https://markist.xyz/api/mcp`.
- **Other MCP clients**: add a Streamable HTTP server at `https://markist.xyz/api/mcp`; the client will prompt for OAuth.

Details and troubleshooting: `https://markist.xyz/guide#mcp`.

## The five skills

| Skill | What it does | Tools |
|---|---|---|
| [`markist-research`](markist-research/SKILL.md) | Searches the user's bookmarks on a topic and answers questions from what's saved, with citations. Read-only. | `search_bookmarks`, `list_tags`, `get_bookmark`, `read_bookmark_content`, `list_shared_with_me`, `explore_collections`, `get_collection` |
| [`markist-capture`](markist-capture/SKILL.md) | Saves links found in the conversation (sources from research, pasted URLs, a reading list) to the library. | `list_tags`, `add_bookmark` |
| [`markist-review`](markist-review/SKILL.md) | A digest of what was saved recently, and triage of the read-later queue (keep / mark read / delete). | `get_insights`, `search_bookmarks`, `mark_bookmark_read`, `delete_bookmark` |
| [`markist-curate`](markist-curate/SKILL.md) | Builds a Collection on a topic from the user's bookmarks, with drafted curator notes. | `search_bookmarks`, `list_my_collections`, `get_collection`, `create_collection`, `add_to_collection`, `set_curator_note` |
| [`markist-share`](markist-share/SKILL.md) | Shares a bookmark or Collection with specific people, or publishes a Collection. | `list_contacts`, `share_bookmark`, `add_share_recipients`, `add_collection_recipients`, `set_collection_visibility`, `remove_share_recipient`, `remove_collection_recipient` |

`markist-capture`, `markist-review`, `markist-curate` and `markist-share` write to the
account, so each carries a copy of the **confirmation protocol**
(`_shared/confirmation-protocol.md`): plan → explicit yes → act → per-item report,
with separate tiers for ordinary writes, destructive actions, and anything that shares or
emails someone. `markist-research` is read-only and carries no such copy.

## Install

Get the skills from [`pragmatico/markist-skills`](https://github.com/pragmatico/markist-skills):

```bash
git clone https://github.com/pragmatico/markist-skills.git
cd markist-skills
```

### Claude Code

Symlink or copy a skill folder into `~/.claude/skills/` (or a project's own
`.claude/skills/`):

```bash
ln -s "$(pwd)/markist-research" ~/.claude/skills/markist-research
# repeat for markist-capture, markist-review, markist-curate, markist-share
```

### Claude.ai / Claude Desktop

Each [release](https://github.com/pragmatico/markist-skills/releases/latest) attaches one
zip per skill (`markist-research.zip`, …), with the skill folder's contents at the zip
root. In Claude.ai or Desktop, go to Settings → Capabilities → Skills → Upload skill, and
upload each zip you want.

To build a zip yourself, zip a skill folder's *contents* (not the folder itself):

```bash
(cd markist-research && zip -r ../markist-research.zip .)
```

### Other agents (Codex, Gemini CLI, Cursor, Copilot, …)

Any agent that reads the open `SKILL.md` format can use these directly — copy the
relevant `<name>/` folder into that agent's own skills directory. Tool calls are
made by bare MCP name (e.g. `search_bookmarks`); a client that prefixes tool names (Claude
Code uses `mcp__markist__search_bookmarks`) resolves the prefix itself, so the skill text
never hard-codes one.

## Development

These skills are developed alongside the Markist server and mirrored to this repository,
so every change is validated there before it lands here: frontmatter shape and limits,
that `references/` links resolve, that each `references/confirmation-protocol.md` copy is
byte-identical to `_shared/confirmation-protocol.md`, and that every MCP tool a skill
names still exists on the server.

### Releasing

Push a bare `vX.Y.Z` tag to this repository. `.github/workflows/skills-release.yml` zips
every skill and attaches the zips to a GitHub Release.
