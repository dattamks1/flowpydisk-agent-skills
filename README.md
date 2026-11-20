# FlowpyDisk agent skills

[FlowpyDisk](https://flowpydisk.com), a product by Invrsys, is a workspace for
files, notes, tables, calendar, meetings, chat, mail and AI search, in one place.

This repository holds the FlowpyDisk agent skill: how an AI agent should use
the FlowpyDisk MCP server. It says which tool to call, how to call it, what
comes back, the limits and error codes, and what an agent must never do.

## What is in it

```
skills/flowpydisk/
  SKILL.md           start here: the whole server in one document
  guides/            one guide per surface (files, notes, tables, canvas,
                     forms, calendar and tasks, booking, mail, search and more)
    README.md        which guide covers which tool
```

`SKILL.md` is a Claude Agent Skill. Any other agent can read it as plain
Markdown. The server also serves the same guides at run time with the
`get_guide` tool.

## Install the skill

### Claude Code: copy the folder

For every project on your machine:

```bash
git clone https://github.com/dattamks1/flowpydisk-agent-skills.git
mkdir -p ~/.claude/skills
cp -R flowpydisk-agent-skills/skills/flowpydisk ~/.claude/skills/
```

For one project only, copy it into that project's `.claude/skills/` folder
instead. Claude Code picks the skill up the next time it starts.

### Claude Code: install as a plugin

The repository is also a plugin marketplace. The plugin adds the skill and the
FlowpyDisk MCP server in one step:

```
/plugin marketplace add dattamks1/flowpydisk-agent-skills
/plugin install flowpydisk@flowpydisk
```

### Other agents

Give the agent `skills/flowpydisk/SKILL.md` as context or a system prompt,
and the guide for the surface it works on from `skills/flowpydisk/guides/`.

## Connect the FlowpyDisk MCP server

The hosted server is at:

```
https://mcp.flowpydisk.com/mcp
```

It signs in with OAuth. No API key is needed. When you connect, FlowpyDisk
asks you to sign in, then to choose which spaces the app may use (your
personal space and any workspaces) and, for each one, read or read and write.
An app can never do more than you can do yourself in the web app. You can
change or remove a connection later in the app under **Settings > Developer**.

**Claude (web and desktop).** Add the address above as a custom connector,
then sign in when asked.

**Claude Code.**

```bash
claude mcp add --transport http flowpydisk https://mcp.flowpydisk.com/mcp
```

Then run `/mcp` in Claude Code and sign in. The plugin above adds the same
server for you.

**Other MCP clients.** Add the address as a remote (Streamable HTTP) MCP
server. The client registers itself and sends you to sign in.

Removals and changes other people rely on (trashing items, sending mail,
answering bookings and similar) ask for your approval in the app before they
run. The skill explains how an agent handles that.

## Scripts and the API

For scripts and server-side code, use an API key with the Python SDK:
[dattamks1/flowpydisk-python-sdk](https://github.com/dattamks1/flowpydisk-python-sdk)
(`pip install flowpydisk`). Make a key in the app under
**Settings > Developer**.

## Contributions

This repository does not accept contributions, pull requests or issues for now.

## License

Apache-2.0. See [LICENSE](LICENSE).
