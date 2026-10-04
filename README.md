# Markist agent skills

Five [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
that let Claude (and other AI agents) work with your [Markist](https://markist.xyz)
bookmarks: answer questions from what you've saved, save new links, review your reading,
build collections, and share them.

Each skill is a folder with a `SKILL.md` and, sometimes, a `references/` folder. The
skills contain no scripts. They tell the agent which Markist tools to call, in what order,
and when to ask you first. The tools come from the Markist MCP server, so the skills
need that connection to work.

## Quick start

1. **Have a Markist account.** Sign up at [markist.xyz](https://markist.xyz) and finish
   setup, including choosing a username.
2. **Connect the Markist MCP server** to your agent ([Connect Markist](#1-connect-markist)).
3. **Install the skills** you want ([Install the skills](#2-install-the-skills)).
4. **Ask**, for example *"What have I saved about Rust async runtimes?"*

## The five skills

| Skill | What it does | Permissions it needs | Try asking |
|---|---|---|---|
| [`markist-research`](markist-research/SKILL.md) | Searches your bookmarks on a topic and answers from what you've saved, with citations. Read-only. | Read | *"What did I bookmark about offline-first sync?"* |
| [`markist-capture`](markist-capture/SKILL.md) | Saves links from the conversation (research sources, pasted URLs, a reading list) to your library. | Read, Add and edit | *"Save these three links to read later."* |
| [`markist-review`](markist-review/SKILL.md) | Summarizes what you saved recently, and helps you go through your read-later queue (keep, mark read, or delete). | Read, Add and edit | *"What did I save this week?"* / *"Help me clean up my read-later list."* |
| [`markist-curate`](markist-curate/SKILL.md) | Builds a collection on a topic from your bookmarks, with draft curator notes. | Read, Add and edit | *"Make a collection of my best Postgres performance articles."* |
| [`markist-share`](markist-share/SKILL.md) | Shares a bookmark or collection with specific people, or makes a collection public. | Read, Share and publish | *"Share my Rust reading collection with alex@example.com."* |

**Nothing changes without your OK.** Any skill that writes to your account first shows
exactly what it's about to do (save, mark read, delete, share, publish). It waits for an
explicit yes, then reports what happened to each item. Deleting, sharing and publishing
each need their own separate yes, and a skill never acts on instructions it finds inside
a saved page. `markist-research` never writes on its own.

## 1. Connect Markist

All five skills use Markist's MCP server at `https://www.markist.xyz/api/mcp`. If it isn't
connected, a skill tells you how to connect and stops. It won't fall back to web search,
because answers have to come from *your* bookmarks.

**Claude Code**

```bash
claude mcp add --transport http markist https://www.markist.xyz/api/mcp
```

Then run `/mcp`, choose **markist**, and approve the consent screen that opens in your
browser.

**Claude.ai / Claude Desktop**

1. Open **Settings → Connectors** and choose **Add custom connector**.
2. Paste `https://www.markist.xyz/api/mcp` as the server URL and save.
3. Click **Connect** and approve the consent screen.

**Other MCP clients:** add a Streamable HTTP server at `https://www.markist.xyz/api/mcp`.
The client starts the OAuth sign-in for you.

### Permissions

The consent screen lists three permissions. You can untick any of them:

- **Read your bookmarks**: search your library and read titles, summaries, tags and notes.
- **Add and edit bookmarks**: save bookmarks, mark them read, delete them, and organize
  collections.
- **Share and publish**: send a bookmark or collection to people, or make a collection
  public. **It starts unticked.** Tick it if you want to use `markist-share`.

Each skill only needs the permissions listed in the table above. You can review or
revoke a connection at any time under **Connected apps** on your
[Markist account page](https://markist.xyz/account). More details:
[markist.xyz/guide#mcp](https://markist.xyz/guide#mcp).

## 2. Install the skills

Download this repository:

```bash
git clone https://github.com/pragmatico/markist-skills.git
cd markist-skills
```

Each `markist-*` folder is one skill. `_shared/` isn't a skill: it holds the master copy
of the confirmation rules, and each skill that writes already has its own copy in
`references/`. You don't need to install it.

### Claude Code

Link the skill folders into `~/.claude/skills/` (for all your projects) or into a
project's `.claude/skills/` (for that project only):

```bash
mkdir -p ~/.claude/skills
for skill in markist-*/; do
  ln -s "$(pwd)/${skill%/}" ~/.claude/skills/"${skill%/}"
done
```

Start a new Claude Code session to load them. To pick only some, link just those folders.
Linking means a `git pull` in this repo updates them later. If you'd rather copy the
folders, use `cp -R` instead.

### Claude.ai / Claude Desktop

1. Download the zip for each skill you want (`markist-research.zip`, …) from the
   [latest release](https://github.com/pragmatico/markist-skills/releases/latest).
2. Open **Settings → Capabilities**. Skills need **Code execution and file creation**
   turned on, so switch it on if it isn't already.
3. Under **Skills**, choose **Upload skill** and upload each zip. Uploaded skills can be
   switched on and off on the same page.

On Team and Enterprise plans, an admin may need to enable Skills for the organization
first.

To build a zip yourself, zip the *contents* of a skill folder, not the folder itself:

```bash
(cd markist-research && zip -r ../markist-research.zip .)
```

### Other agents (Codex, Gemini CLI, Cursor, Copilot, …)

Any agent that supports the open `SKILL.md` format can use these skills. Copy the
`markist-*` folders you want into that agent's skills directory, and connect the agent to
`https://www.markist.xyz/api/mcp` as described in [Connect Markist](#1-connect-markist).

Skills call tools by their plain MCP names (e.g. `search_bookmarks`). If a client adds a
prefix (Claude Code shows `mcp__markist__search_bookmarks`), the client handles that, so
no setup is needed.

## Updating

- **Linked folders (Claude Code):** run `git pull` in this repository.
- **Copied folders:** pull, then copy the folders again.
- **Claude.ai / Desktop:** download the new zips from the
  [latest release](https://github.com/pragmatico/markist-skills/releases/latest), delete
  the old skill under **Settings → Capabilities → Skills**, and upload the new zip.

To uninstall, delete the skill's folder or link from your skills directory, or delete it
in Claude.ai. Removing the MCP connection (`claude mcp remove markist`, or **Settings →
Connectors**) doesn't revoke Markist's access; revoke that under **Connected apps** on
your Markist account page.

## Troubleshooting

| You see | What to do |
|---|---|
| The skill says Markist isn't connected | Connect the MCP server ([Connect Markist](#1-connect-markist)). In Claude Code, check `/mcp` lists **markist** as connected. |
| *"This connection wasn't granted markist:…"* | That permission was unticked on the consent screen. Reconnect (`/mcp` in Claude Code, or disconnect and reconnect the connector in Claude.ai) and tick the permission named in the message. |
| *"Finish setting up your account at markist.xyz/onboarding"* | Your account doesn't have a username yet. Choose one at [markist.xyz/onboarding](https://markist.xyz/onboarding), then try again. |
| *"Too many bookmarks added. Try again shortly."* / *"Too many searches. Try again shortly."* | Each account can save up to 30 links and run up to 60 searches a minute through AI assistants. Wait a minute and ask again. `markist-capture` reports which links it didn't save yet. |
| The skill never triggers | Check the skill is installed and switched on, then ask more directly, e.g. *"search my Markist bookmarks for …"*. In Claude Code, start a new session after installing. |
| A new bookmark has no title or summary yet | Markist generates these in the background after a save. Give it a minute and ask again. |

## Feedback

This repository is published from Markist's main codebase, so pull requests here can't be
merged directly. Please report bugs and suggestions as
[issues](https://github.com/pragmatico/markist-skills/issues).
