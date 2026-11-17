# Finding a past Claude conversation

How to resolve a Claude or Claude Code conversation title to the real CLI
session ID and transcript, including conversations run on a remote machine
through Claude Desktop's SSH support.

Placeholders used below:

- `<ssh-host>`: the SSH host alias of your remote dev machine.
- `<remote-home>`: your home directory on that machine, e.g. `/home/<user>`.

## 1. Start with Claude Desktop's session metadata

On a Mac, Claude Desktop keeps one small JSON metadata file per conversation
under:

```text
~/Library/Application Support/Claude/claude-code-sessions/**/local_<id>.json
```

This is the best title index. In particular, a user-assigned display title can
exist here without appearing anywhere in the transcript itself. Search this
directory first, using the exact title and returning paths rather than whole
matching files:

```bash
rg -l -F --hidden \
  "<exact conversation title>" \
  "$HOME/Library/Application Support/Claude/claude-code-sessions"
```

Inspect only the useful fields from the matching metadata file:

```bash
jq '{
  title,
  titleSource,
  previousTitles,
  cliSessionId,
  cwd,
  originCwd,
  model,
  effort,
  isArchived,
  sshConfig,
  sshRemoteTranscriptPath,
  forkedFromSessionId
}' '<metadata-file>.json'
```

Important distinctions:

- `title` is the Claude Desktop display title and is the authoritative field
  when the user supplies an exact conversation name.
- `previousTitles` holds earlier names of a renamed conversation, so a search
  for an old title can still land on the right file.
- `cliSessionId` is the ID to pass to `claude --resume`.
- `sessionId` in this metadata often has a `local_` prefix; do not pass that
  prefixed value to the CLI.
- `cwd` is the directory from which the session should be resumed.
- For an SSH conversation, `sshConfig.sshHost` identifies the host and
  `sshRemoteTranscriptPath` gives the exact transcript path on that host.
- `forkedFromSessionId` helps distinguish copies and branches that share most
  of their history.

## 2. Locate a local CLI transcript

Claude Code stores local transcripts under:

```text
~/.claude/projects/<cwd-with-slashes-encoded>/<cli-session-id>.jsonl
```

For example, a conversation started in `/Users/example/projects/my_repo` will
normally be under a directory resembling:

```text
~/.claude/projects/-Users-example-projects-my-repo/
```

Once the metadata gives you the CLI session ID, find the transcript directly:

```bash
find "$HOME/.claude/projects" \
  -type f -name '<cli-session-id>.jsonl' -print
```

Some CLI-authored titles are also copied into the transcript as
`custom-title` records. This is a useful fallback when no Claude Desktop
metadata is available:

```bash
jq -r '
  select(.type == "custom-title") |
  [.customTitle, .sessionId] |
  @tsv
' '<transcript>.jsonl' | sort -u
```

A transcript can contain repeated title records or titles inherited from a
fork. Prefer the exact Claude Desktop metadata match when it exists.

## 3. Locate an SSH transcript on a remote machine

Claude Desktop's local metadata usually tells you everything needed:

```json
{
  "cliSessionId": "<uuid>",
  "cwd": "<remote-home>",
  "sshConfig": {"sshHost": "<ssh-host>"},
  "sshRemoteTranscriptPath": "<remote-home>/.claude/projects/<encoded-cwd>/<uuid>.jsonl"
}
```

Verify that exact remote file instead of scanning the whole machine:

```bash
ssh <ssh-host> 'stat "<sshRemoteTranscriptPath>"'
```

The display title may be absent from both the remote transcript and
`~/.claude/history.jsonl`; it can live only in the local Claude Desktop
metadata. This is why searching the whole remote machine for the exact title
can fail even though the conversation is present.

If the Desktop metadata is unavailable, search remote transcripts by a
distinctive phrase from the conversation rather than by its display title:

```bash
ssh <ssh-host> \
  'rg -l -F --hidden "<distinctive prompt phrase>" \
    "$HOME/.claude/projects" --glob "*.jsonl"'
```

Multiple files can share the same first prompt when they are parallel forks.
Do not choose among them by size or modification time alone. Inspect fork
metadata, later user messages, or obtain the Desktop title mapping.

Local mirror directories such as `~/.claude/projects/ssh-<cli-session-id>/`
may also exist for SSH sessions, but they are not a reliable display-title
index.

## 4. Resume the conversation

For a local conversation, run from the metadata's `cwd`:

```bash
cd '<cwd>'
claude --resume '<cliSessionId>'
```

For a remote conversation:

```bash
ssh -t <ssh-host> \
  'cd "<cwd>" && claude --resume "<cliSessionId>"'
```

## 5. Read transcripts selectively

Claude transcripts are mixed-content JSONL containing messages, tool calls,
tool results, progress records, and metadata. Do not `cat` a multi-megabyte
transcript into an agent context. Extract only the conversation text first:

```bash
jq -r '
  select(.type == "user" or .type == "assistant") |
  .type as $role |
  .message.content |
  if type == "string" then
    "=== " + ($role | ascii_upcase) + " ===\n" + .
  elif type == "array" then
    [.[] | select(.type == "text") | .text] |
    select(length > 0) |
    "=== " + ($role | ascii_upcase) + " ===\n" + join("\n")
  else
    empty
  end
' '<transcript>.jsonl'
```

If exact commands, patches, or tool output matter, inspect only the matching
raw JSONL records after narrowing with a distinctive phrase. As with Codex
transcripts (see [codex_conversation_lookup.md](./codex_conversation_lookup.md)),
the assistant's prose summary is not a substitute for the actual tool-call
payload or resulting file.

## 6. Avoid the common dead ends

- Do not begin with `rg` across the entire home directory. Large repositories,
  caches, and generated artifacts make it slow and produce false positives.
- Do not assume the display title must appear in the transcript.
- Do not confuse a mention of the requested title in your current task's own
  context with the old conversation being sought.
- Use `rg -l` and `-F` for exact title searches so large JSON or JSONL records
  are not printed to the terminal.
- When several sessions share an opening prompt, resolve the title through
  Claude Desktop metadata before selecting a transcript.
