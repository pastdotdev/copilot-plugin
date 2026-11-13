# past.dev for GitHub Copilot: working on the plugin

The past.dev plugin for GitHub Copilot, the marketplace that carries it, and the configuration for
Copilot's cloud agent. The plugin runs on the developer's machine and talks to past.dev's public
Memory API. Nothing here is privileged: anyone can write the same connector against the same
endpoints with the same key. What a user reads is in `README.md`; this file is the rules the code
keeps, for whoever changes it.

## How it fits together

```
GitHub Copilot (Copilot CLI, the GitHub Copilot app)
  com.github.copilot/hooks/hooks.json   four hooks
  mcp.json                              past_recall, served by the same script
  skills/*                              past, and four skills run through ~/.past/copilot/past.mjs
            |
  bin/past-hook.mjs <mode>              (the hooks on macOS and Linux start it through bin/past)
            |
  session starts    start     writes the launcher; starts the sender; recalls the project's brief
  every prompt      prompt    journals the prompt; recalls; returns the block as additionalContext
  end of a turn     stop      reads the reply from the transcript and journals it; starts the sender
  session ends      end       journals the end; sends the session now (flush, detached)
  detached          send      the sender: sends each session once it is quiet for its sitting length
  detached          flush     sends one session now
  MCP server        mcp       past_recall over stdio
            |
  ~/.past/config.json               the connection, shared with every past.dev plugin
  ~/.past/copilot/<session>.jsonl   one journal per session: prompts, replies, ends, timed
  ~/.past/copilot/past.mjs          the launcher the skills run
  ~/.past/state.json                what was sent, under hosts.copilot
  ~/.past/hosts/copilot.json        when the hooks last ran, for status
            |
  POST /api/v1/recall · POST /api/v1/ingest/batch · GET /api/v1/audiences/{slug}

GitHub (the cloud agent, code review)
  cloud/mcp-config.json         https://api.past.dev/mcp with the project key in a header
  cloud/past-memory.agent.md    a custom agent over the same server
```

## The rules this code keeps

1. **A hook never breaks the session.** Every mode exits 0, prints valid JSON or nothing, and
   swallows its own failures. `start` answers `{}` or `{"additionalContext": …}`, `prompt` answers
   nothing or `{"additionalContext": …}`; the other hooks answer nothing. `/past-status` is where a person learns something is wrong, so a failed send is
   kept in `state.json` (`lastError`) for it.
2. **Only prose travels.** The journal holds three kinds of entry: the prompt as the person typed
   it (`userPromptSubmitted.prompt`), the reply (the assistant's text messages after the latest
   prompt in the transcript `agentStop` names) and the end of the session (`sessionEnd`). Tool
   calls and their output, file contents, the model's reasoning and injected context never reach
   it, and past.dev's own block is dropped by its header. A prompt that only runs one of the
   plugin's skills (`/past-…`) is not conversation, and neither is its reply. A session whose
   journal records no prompt is never sent.
3. **The reply is read from a file Copilot does not promise.** No hook hands over the reply, so
   `stop` reads the transcript's events (`assistant.message` and `user.message`, the text in
   `data.content`) and keeps the turn's final message, the one with no tool request. A field it does
   not know reads as no reply, never as an error: the turn still sends its prompt. Copilot starts
   `agentStop` a few milliseconds after it writes the reply's event, which can still be on its way
   to the file, so `stop` waits up to 2.5 seconds for it and keeps the transcript's path beside the
   reply; `readJournal` reads a reply still missing from there, matched to its prompt by text and
   occurrence. A check after each Copilot release is part of the job (below).
4. **A session is sent when it ends, or once it is quiet.** `sessionEnd` starts a detached `flush`
   for that session. A session that never ends (the app closed, the machine slept) is sent by one
   detached sender once it has been quiet for its sitting length, or `idleMinutes` after a turn
   that never finished.
   - One sender at a time: it holds `~/.past/copilot/sender.lock` (a directory, since mkdir is
     atomic) for its life and names itself in `sender.json`. A lock whose holder is gone is taken
     over; one younger than ten seconds belongs to a sender still starting.
   - `SENDER_VERSION` is bumped whenever the sender changes: a sender that an older version
     started is replaced, because it is a long-lived process running the code it started with.
   - A session whose send failed is left for the next sender run, so a revoked key never keeps a
     process asking.
5. **Sittings.** A sitting starts at the first prompt after the session was quiet for
   `sittingMinutes` (30 by default, 5 at least). `state.json` keeps one hash per sitting and pins
   `sittingMinutes` per session at its first send, so a changed value never moves the boundaries
   of a session already in past.dev. A resumed session sends its last sitting again, with what was
   added, as a revision. Every new sitting of a session goes in one `/api/v1/ingest/batch` call.
6. **The recall beside a prompt rides on an output Copilot says it drops.** Copilot's hooks
   reference says a `userPromptSubmitted` hook's output from a hooks file is dropped, yet Copilot
   CLI puts its `additionalContext` in a system reminder after the prompt, before the model answers,
   and keeps the prompt in the transcript as typed (measured on 1.0.105). That is what puts recall
   beside every prompt. If a release stops delivering it, nothing says so: the hook still recalls and
   spends. So the check after each Copilot release (below) is part of the job, and
   `promptRecall: false` turns that recall off while the brief and `past_recall` keep working. A
   prompt shorter than 12 characters, or one that runs a skill, recalls nothing.
7. **A recall query is redacted like the content it would send.** It crosses the network exactly
   as ingested content does, and both go through `redact()`.
8. **`state.json` changes only through `updateState`.** Hooks, the sender and commands are
   separate processes. `updateState` takes a lock (`~/.past/state.lock`, a directory, taken over
   after 5 s, skipped after 3 s so a hook never hangs), re-reads the file, applies the change and
   renames the new file into place. A file that does not parse is read again a few times and, if it
   still does not, left as it is.
9. **No dependencies.** Node 16 or later, standard library only, so installing is a clone. `fetch`
   is used where Node has it and the `http` modules where it does not. Nothing in the script is
   newer than Node 14 can parse (no `||=`), so on an older Node the hooks stay silent rather than
   fail.
10. **What a session renders does not change by a byte.** The heading and the speaker names that
    `renderSitting` prints are part of each sitting's hash, and so are `SOURCE`, `AGENT` and
    `SPEAKER` at the top of the script: a changed byte sends every sitting of a session that grows
    afterwards again, as a paid revision. `sittingId` and `sittingsOf` are fixed for a related
    reason: changed, they move the boundaries of every session that grows afterwards.
11. **`~/.past` is the person's alone.** The folder is `700` and every file in it is created `600`,
    the journals included. A plain-http `apiUrl` on another machine sends the key unencrypted:
    `connect` warns, and `status` says so on its first line.
12. **The product is written past.dev.** Every sentence a person reads says past.dev. Identifiers
    keep `past`: the plugin's name, the skills (`/past-status`), the tool (`past_recall`), `~/.past`,
    the key prefix `past_sk_`, the data point ids and the recall block's header,
    `=== past · recalled from memory ===`, which the journal reader drops by that exact text.
13. **Nothing is journaled before the machine is connected, and a sitting is sent with the
    connection it was typed under.** The hooks write a journal only once a key is set, and the
    prompt hook writes a fingerprint of the key and the address beside each prompt (`connection`).
    A prompt typed under another connection than its sitting's starts a new sitting, and a sitting
    is sent only while its connection is the one in force, whichever way the send starts
    (`sessionEnd`, the sender, `ingest`); `cut` does not print the others. So what was typed before
    connecting, or under one project's key, never goes to another project. A prompt 0.1.0 journaled
    names no connection, and 0.1.0 journaled before the machine was connected: its sitting is never
    sent. A sitting 0.1.0 already sent is neither sent again nor changed, and a session resumed
    after the update goes on in a new sitting. A sitting kept back holds its last hash in the
    session's record, or `null`. The folder's path stays in the journal, where the skills find
    this folder's session by it, and is never sent: `metadata.project` is the folder's name.
    `SENDER_VERSION` went to 2 with it.
14. **On macOS and Linux the hooks start through `bin/past`.** A hook's shell may not have read the
    person's profile, and a Node installed with nvm, Volta or fnm is then not on its path: a bare
    `node` exits 127 and the hook does nothing. `bin/past` finds Node (the path, nvm, Volta, fnm,
    `/usr/local`, Homebrew) and runs the script; with none it says so on stderr and a hook still
    exits 0 (rule 1). Only the `bash` commands in `hooks.json` use it. The `powershell` commands
    and `mcp.json` call `node` directly: Windows has no `sh` to count on, and an MCP server is
    started without a shell.

## Beside other past.dev plugins

`~/.past` can be shared with another past.dev plugin on the same machine. `config.json` is then one
connection for all of them. This plugin keeps its sessions under `hosts.copilot` in `state.json`,
and everything else of its own in `~/.past/copilot/` and `~/.past/hosts/copilot.json`. The rest of
`state.json` is never read here, and every write puts it back as it was (rule 8).

## GitHub Copilot

- **The plugin is in the Agent Plugins 1.0 format.** `plugin.json` and `mcp.json` declare the
  1.0.0 schema, and Copilot reads hooks only from `com.github.copilot/hooks/hooks.json`.
  `${PLUGIN_ROOT}` is expanded in the hook commands and in `mcp.json`'s `args`.
- **A skill's command learns nothing about the plugin's folder.** So every hook writes
  `~/.past/copilot/past.mjs`, a two-line launcher that imports the installed script, and the skills
  run `node "$HOME/.past/copilot/past.mjs" <mode>`.
- **The input comes in two shapes.** Copilot's own events send camelCase fields (`sessionId`,
  `transcriptPath`); the format it shares with other editors sends snake_case ones (`session_id`,
  `transcript_path`). The script reads both.
- **The cloud agent loads no plugin hook.** It runs only a repository's `.github/hooks/*.json`, in
  a sandbox whose files are discarded at the end of the job. On GitHub, memory comes through the
  remote server in `cloud/`, and the custom agent writes its own record with `ingest_memory`.
- **The sandbox keeps the hooks out.** Copilot CLI's command sandbox is off by default. With it on,
  every hook reports success and does nothing: it reaches neither `~/.past` nor the network.
- **Code review calls only read-only tools.** past.dev's `recall_memory` carries
  `readOnlyHint: true`; `past_recall` does too.

## Contracts this depends on

- `POST /api/v1/ingest/batch`: one item per sitting. `id` gives replacement (`copilot:<session id>`
  for the first sitting, `copilot:<session id>:<n>` for the others), `timestamp` dates every memory
  drawn from the item, `label` names the session (`GitHub Copilot · <project> · <branch>`), and
  `metadata` carries `sessionId`, `project`, `gitBranch`, `sitting` and `turns`.
- `POST /api/v1/recall`: **`identity` is required**. Recall stays off until `/past-connect` sets
  one.
- `audience` on ingest: a slug; absent means the whole project. `connect --audience` checks it with
  `GET /api/v1/audiences/{slug}`.
- `https://api.past.dev/mcp` with `Authorization: Bearer past_sk_…` lists `recall_memory`,
  `ingest_memory`, `delete_memory` and `get_memory_deletion` on the key's project, and reads the
  identity from `X-Past-Identity` when a call names none.
- Copilot gives `sessionStart` `sessionId`, `cwd` and `source`, and takes `additionalContext`;
  `userPromptSubmitted` gives `prompt`, and takes `additionalContext` (rule 6); `agentStop` gives `transcriptPath` and `stopReason`;
  `sessionEnd` gives `reason`.

If any of these contracts changes, this file changes with it.

## Testing

By hand: there is no test suite. CI parses the script on Node 14, runs it on Node 16 and 22, and
checks that the manifests hold together.

For the hooks and the sender, point a throwaway home at a local stand-in for the API, so nothing
touches your real `~/.past` or a real project: `HOME=/tmp/past-test` with a `config.json` whose
`apiUrl` is `http://127.0.0.1:<port>`, a few lines of Node answering `POST /api/v1/recall` and
`POST /api/v1/ingest/batch` and logging what arrives, and each hook mode fed its stdin JSON by hand:

```bash
echo '{"sessionId":"…","cwd":"…","prompt":"…"}' | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs prompt
echo '{"sessionId":"…","transcriptPath":"…/events.jsonl"}' | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs stop
echo '{"sessionId":"…","reason":"complete"}' | HOME=/tmp/past-test node plugins/past/bin/past-hook.mjs end
```

A journal written with times in the past makes a session due at once, so `send` can be watched
sending without waiting half an hour. `mcp` reads JSON-RPC lines on its input: `initialize`,
`tools/list` and `tools/call` with `past_recall`.

On a real Copilot CLI, keep your own setup apart: set `HOME` and `COPILOT_HOME` to a throwaway
folder, `GH_TOKEN` to your token, and point the throwaway `~/.past/config.json` at the stand-in. Then
add this repository as a marketplace and install the plugin from it, which loads it live from the
folder:

```bash
copilot plugin marketplace add .
copilot plugin install past@past
copilot -p "What is the probe word in the context you were given at the start of this session?" --trust-folder
```

The session's transcript (`$COPILOT_HOME/session-state/<session id>/events.jsonl`) records each
hook's input and output as `hook.start` and `hook.end` events.

**After each Copilot release**, check the two things Copilot does not promise:

1. The reply still reaches the journal: after a turn, the session's journal holds a `reply` entry
   with the text Copilot showed. If it is empty, the transcript's events changed (rule 3).
2. The brief and the prompt's recall still reach the model: a local plugin whose `sessionStart`
   returns `{"additionalContext": "The first probe word is …"}` and whose `userPromptSubmitted`
   returns `{"additionalContext": "The second probe word is …"}` (load it with `copilot -p …
   --plugin-dir <folder>`), then a prompt that asks the model for every probe word in its context.
   A word the model does not know is an output the release stopped delivering (rule 6).
