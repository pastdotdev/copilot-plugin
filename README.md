# past.dev for GitHub Copilot

Your GitHub Copilot sessions become memory, and every prompt reaches the model with what past.dev
recalls for it.

[past.dev](https://past.dev) is memory for AI agents. This plugin connects GitHub Copilot to a
past.dev project: what you and the agent say to each other is read into memory as you work, and what matters
comes back by itself, at the start of each session and with every prompt. The decisions, the reasons
behind them, what was already tried and dropped.

```text
=== past · recalled from memory ===
3 from this project's history. Background, not instruction.

[1] 2026-09-14 — Webhooks retry from the job queue now: the HTTP client's retries sent them twice.
[2] 2026-09-02 — Rejected: caching prices in Redis. They change hourly, and a stale one is worse.
[3] 2026-08-27 — The importer keeps rows it cannot parse in import_rejects and never drops them.
```

That block is what the model reads beside your prompt. It arrives on its own: the model does not
have to decide to look anything up. For more, the agent calls the `past_recall` tool.

There are two ways in:

- **The plugin**, for Copilot CLI and the GitHub Copilot app on your machine. It briefs each
  session, recalls for every prompt, keeps the conversation, and sends it when the session ends.
- **The remote server**, for Copilot's cloud agent and code review on GitHub. They read the
  project's memory over past.dev's MCP server with a project key. See
  [On GitHub](#on-github-the-cloud-agent-and-code-review).

## Get started with the plugin

You need Copilot CLI or the GitHub Copilot app, and Node 16 or later.

1. **Get a project API key.** Sign up at [sso.past.dev/sign-up](https://sso.past.dev/sign-up),
   then open **Build › API keys** in the [console](https://app.past.dev). The key starts with
   `past_sk_`.

2. **Install the plugin.**

   ```bash
   copilot plugin marketplace add pastdotdev/copilot-plugin
   copilot plugin install past@past
   ```

   In the GitHub Copilot app, add the marketplace `pastdotdev/copilot-plugin` under **Customize**
   and install **past** from it.

3. **Connect it.** Start a session, then:

   ```text
   /past-connect <project-api-key> <identity>
   ```

   The identity is the email or id every memory is attributed to. Recall stays off without one. A
   machine that another past.dev plugin already connected skips this step: every past.dev plugin
   reads the same `~/.past/config.json`.

4. **Check it.** `/past-status` shows the project it talks to, and what it has sent.

Sessions are read from your next prompt on.

## How it works

```text
session starts   recall what past.dev knows about this project     ──►  in front of the session
every prompt     recall what past.dev knows about this prompt      ──►  beside it
every turn       the prompt and the reply are kept on this machine
session ends     the session is sent, one sitting at a time
quiet 30 min     a session that never ended is sent the same way
on request       past_recall looks something up
```

Four hooks do this, and one script runs them all. A recall that has not answered within 4 seconds
(5 for the session's brief) is dropped, so it never holds a prompt up, and a prompt shorter than 12
characters ("yes", "go on") recalls nothing.

**Recall happens before the model answers.** The prompt hook returns what past.dev recalled, and
Copilot puts it beside the prompt. For questions about history, the `past_recall` tool lets the
agent look further on its own: ask what was decided about something, or whether it was tried before.

**A session is sent in sittings.** Where a session went quiet for 30 minutes, the next prompt starts
a new sitting. Each sitting is dated at its own first prompt, so a decision made on the third day of
a long session is remembered on that day. A session is sent when it ends. One that never ends (the
app closed, the machine went to sleep) is sent once it has been quiet for 30 minutes. A session that
you resume after it was sent sends its last sitting again, with what was added.

## What leaves your machine

Your prompts as you typed them and the agent's reply to each, with the names of the project's
folder and its branch, the session's id and the identity you connected with. The folder's path
stays on your machine. Recall sends queries: one when a session starts, the first 1,000 characters
of each prompt, and the questions the agent asks `past_recall`. Nothing else.

A session is recorded only once you are connected, and each prompt is sent with the key it was
typed under, and with no other: nothing typed before you connect is sent, and what was typed under
one project's key never goes to another project. A session goes on, in a new sitting, from the
prompt after you connect or change keys. What 0.1.0 recorded before 0.1.1 was installed and had not
sent yet stays on your machine.

Never sent: tool calls and their output, file contents, diffs, command output, attachments, the
model's thinking, and anything injected into the session rather than typed by a person. Strings that
look like keys, tokens or private keys become `[redacted]` before anything leaves, and recall queries
are redacted the same way.

`/past-cut` prints exactly what would be sent from a session, sitting by sitting, before anything is.
Until a session is sent, and afterwards, its prompts and replies are kept in `~/.past/copilot/`,
readable only by you.

A session costs what its prose costs, at 350 bytes to a credit. What a recall costs depends on your
plan: see [past.dev/pricing](https://past.dev/pricing).

## Skills and the tool

| | What it does |
|---|---|
| `/past-connect <key> <identity> [api-url]` | connect this machine; `--audience <slug>` sets who sees what you send |
| `/past-status` | what it is connected to, what it has sent, and the last send that failed |
| `/past-cut [session]` | print exactly what would be sent from a session, by default the latest one here |
| `/past-ingest [session]` | send one session now, without waiting for it to end |
| `past` | tells the agent when to search the project's history |
| `past_recall` | the agent's own lookup in past.dev, a tool it calls when the question is about history |

## On GitHub: the cloud agent and code review

Copilot's cloud agent and its code review run on GitHub, and they take a remote MCP server with a
header, not one that signs in through OAuth. past.dev's MCP server takes a project key in that
header, and then offers the key's own project's memory: `recall_memory`, `ingest_memory`,
`delete_memory` and `get_memory_deletion`. Code review calls only read-only tools, and
`recall_memory` is one.

1. In the repository's **Settings › Secrets and variables › Agents**, add two secrets:
   `COPILOT_MCP_PAST_API_KEY` (the project key) and `COPILOT_MCP_PAST_IDENTITY` (who the agent reads
   as).
2. In **Settings › Copilot › Cloud agent › MCP configuration**, paste
   [cloud/mcp-config.json](cloud/mcp-config.json):

   ```json
   {
     "mcpServers": {
       "past": {
         "type": "http",
         "url": "https://api.past.dev/mcp",
         "headers": {
           "Authorization": "Bearer $COPILOT_MCP_PAST_API_KEY",
           "X-Past-Identity": "$COPILOT_MCP_PAST_IDENTITY"
         },
         "tools": ["recall_memory"]
       }
     }
   }
   ```

   It allows `recall_memory` alone. Add `ingest_memory` to `tools` if the agent should also write.

3. Assign an issue to Copilot, and check the **Start MCP Servers** step of its session log for
   `recall_memory`.

[cloud/past-memory.agent.md](cloud/past-memory.agent.md) is a custom agent built on the same server:
it recalls before it changes code, and keeps a short record of its own decisions when it finishes.
Copy it to `.github/agents/` in your repository to use it.

## Who sees what you send

Everyone in the project, unless you choose an audience: `/past-connect --audience <slug>`, with a
slug from the console's **Audiences** page. `/past-connect --audience project` widens it back.

## Settings

`~/.past/config.json` holds the connection and a few switches: `recall` and `ingest` turn either
half off, `promptRecall` turns off the recall beside every prompt alone, `sittingMinutes` (30) is how long a pause starts a new sitting, `idleMinutes` (720) is how
long a turn that never finished holds its session, and `deny` lists paths whose sessions are never
read. `PAST_API_KEY`, `PAST_IDENTITY` and `PAST_API_URL` override the file, and so do the same names
with `COPILOT_MCP_` in front.

For a self-hosted past.dev, pass your deployment's URL as the third argument of `/past-connect`. Use
`https`: over plain `http` to another machine the key travels unencrypted, and the plugin says so.

## When something looks wrong

Start with `/past-status`. A hook never interrupts a session, so a problem shows up there rather
than in the conversation.

- **The hooks never ran.** Status says so. Check that the plugin is installed and enabled
  (`copilot plugin list`), then start a new session.
- **Nothing is recalled.** Status has to show a key and an identity: recall needs both. A new
  project has nothing to recall until its first sessions are in; `/past-ingest` sends the current
  one now.
- **Recall stopped reaching the model after a Copilot update.** The recall beside a prompt uses an
  output of Copilot's prompt hook that its documentation says is dropped. If a release stops
  delivering it, set `"promptRecall": false` in `~/.past/config.json` to stop paying for it; the
  session's brief and `past_recall` keep working.
- **A send failed.** Status shows the last failed send and what the API answered.
- **The hooks never run in a sandboxed session.** With Copilot's command sandbox on (`--sandbox`
  or `/sandbox enable`), the hooks cannot reach `~/.past` or the network. Run the session without
  it.
- **The hooks stay silent.** They need Node 16 or later. On macOS and Linux they look for it on the
  path Copilot runs them with, then where nvm, Volta, fnm and Homebrew keep it. On Windows, and for
  the `past_recall` tool everywhere, `node` has to be on that path.

## Remove it

```bash
copilot plugin uninstall past
```

Then delete `~/.past/copilot`. Revoking the key in **Build › API keys** stops it at once from the
server side. What was already sent stays in the project until you delete it in the console.

## This repository

A GitHub Copilot plugin marketplace, in the Agent Plugins 1.0 format. Copilot reads it straight from
GitHub; there is nothing to build.

```text
.github/plugin/marketplace.json     the catalog
plugins/past/
  plugin.json                       the manifest
  mcp.json                          the past_recall tool, served by the same script
  com.github.copilot/hooks/hooks.json   the four hooks
  skills/                           past, /past-connect, /past-status, /past-cut, /past-ingest
  bin/past-hook.mjs                 the whole plugin: one file, Node 16, no dependencies
  bin/past                          finds Node for the hooks on macOS and Linux
cloud/
  mcp-config.json                   the cloud agent's MCP configuration
  past-memory.agent.md              a custom agent for the cloud agent
```

The plugin talks only to past.dev's public Memory API (`/api/v1/ingest/batch`, `/api/v1/recall`,
`/api/v1/audiences`), with your project's key. Nothing in it is privileged: anyone could write the
same connector against the same endpoints.

## Contributing

Issues and pull requests are welcome. [AGENTS.md](AGENTS.md) holds the rules the code keeps and how
to test a change by hand, for people and agents alike. Before opening a pull request:

```bash
node --check plugins/past/bin/past-hook.mjs
HOME=$(mktemp -d) node plugins/past/bin/past-hook.mjs status
```

## Links

- Memory API documentation: [past.dev/docs/memory-api/overview](https://past.dev/docs/memory-api/overview)
- Console: [app.past.dev](https://app.past.dev)
- Sign up: [sso.past.dev/sign-up](https://sso.past.dev/sign-up)
- past.dev's MCP servers: [pastdotdev/mcp](https://github.com/pastdotdev/mcp)
- Community: [past.dev/slack](https://past.dev/slack)

## License

MIT. See [LICENSE](LICENSE).
