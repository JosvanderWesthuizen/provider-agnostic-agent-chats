# Finding and mining a past Codex conversation

How to locate a specific local Codex thread by name and pull real context out
of it (not just the chat text) when someone says something like "check my
Codex thread called X".

## 1. Find the thread ID from its title

Codex keeps a flat index of every thread with its title:

```bash
rg -i "<keyword from the thread title>" ~/.codex/session_index.jsonl
```

Each match is `{"id": "...", "thread_name": "...", "updated_at": "..."}`.
Titles aren't unique and get renamed over a thread's life, so a thread can
appear multiple times with different `thread_name` values for the same `id`.
If you know the exact title, search for the most distinctive phrase in it
(skip common words) to avoid dozens of unrelated hits.

## 2. Turn the ID into a transcript file

```bash
find ~/.codex/sessions ~/.codex/archived_sessions -iname "*<thread-id>*"
```

Live threads live under `~/.codex/sessions/YYYY/MM/DD/`; threads Codex
considers closed or archived move to `~/.codex/archived_sessions/`. Check
both. The file is `rollout-<timestamp>-<id>.jsonl`, one JSON object per line,
and is usually zstd-compressed as `rollout-<timestamp>-<id>.jsonl.zst`.

Compressed rollouts are the trap here: plain `grep` and `rg` silently skip
their contents, so a search over `~/.codex/sessions` can come back empty even
though the answer is there. Use `rg -z` to search them, and `zstd -dcf` to
read them (it decompresses `.zst` files and passes plain `.jsonl` files
through unchanged).

## 3. Don't cat the raw file: it's mixed-content JSONL

A real working thread is tens of thousands of lines: reasoning traces, tool
calls, tool outputs, and chat messages all interleaved as different `payload`
types. Loading the whole thing into an agent's context directly is wasteful
and can blow the context window. Extract only what you need:

```bash
zstd -dcf '<rollout file>' | jq -r '
  select(.type == "response_item" and .payload.type == "message") |
  .payload as $p |
  select($p.role == "user" or $p.role == "assistant") |
  ([$p.content[]? | objects | .text // empty] | join("")) as $text |
  select($text | test("\\S")) |
  "=== " + ($p.role | ascii_upcase) + " ===\n" + $text + "\n"
' > /tmp/<thread-id>-chat.txt
```

This pulls just user/assistant chat text (skipping `developer` system
messages and `reasoning`, `function_call`, and `custom_tool_call*` payloads)
into one flat file you can
`grep -n` for keywords, then read with an offset and limit around the matching
line numbers for surrounding context.

## 4. The chat text is a summary, not the source of truth

The biggest trap: what an agent *says* it did in the chat is a paraphrase, not
the actual artifact. If the question is about exact wording (a prompt, a
config, a diff), the ground truth lives in the tool-call and tool-output
payloads, not the chat message. Two ways to get to it:

- **Check if the file still exists on disk.** Agents patch real files in the
  workspace. Run `find <repo> -iname "<filename>"` first. If the file is still
  there, read it directly instead of reconstructing it from chat.
- **If not, search the raw rollout** (not the extracted chat-only file) for
  the distinctive phrase. It will show up inside a `custom_tool_call` (the
  `input` field, e.g. an `apply_patch` payload) or a `patch_apply_end` event
  (the `changes[...].content` field), which has the literal file content at
  the time it was written. That is a better source than the assistant's own
  summary of it.

```bash
# Count matches, including inside compressed rollouts.
rg -z -c -i -F "<exact phrase>" \
  ~/.codex/sessions ~/.codex/archived_sessions --glob "*<thread-id>*"

# Show the record types that contain the phrase, truncated.
zstd -dcf '<rollout file>' |
  rg -i -F "<exact phrase>" |
  jq -c '{type, payload_type: .payload.type, record: (tostring | .[0:3000])}'
```

## 5. Check the workspace for leftovers from past agent runs

Agents that patch files locally before pushing them to a remote machine or a
worktree often leave scratch and staging directories behind (for example
`.codex_tmp*/`, `.codex_*_patch/`, or `*_worktree/`). Run
`find <repo> -iname "<filename>"` before you assume something only exists on
a remote box: it's often sitting right there, uncommitted.

Codex also keeps its own memory notes under `~/.codex/memories/`, which can
point you at the right thread faster than a full-text search.
