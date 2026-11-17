# agent-playbook

Your past AI coding chats aren't locked into one provider. These guides let
any coding agent (Claude Code, Codex, or another) find and read your old
Claude and Codex conversations, so you can switch tools and keep the context.

## What's here

- `AGENTS.md`: tells your agent when to use the two guides below.
- `claude_conversation_lookup.md`: find a Claude or Claude Code conversation
  by its title, then read or resume it (local or over SSH).
- `codex_conversation_lookup.md`: find a Codex thread by its title and pull
  out what was said and what was actually changed.

## Use it

1. Copy the three files into the folder you run your agent from, or paste the
   lines from `AGENTS.md` into your existing one.
2. Codex reads `AGENTS.md` automatically. For Claude Code, add the line
   `@AGENTS.md` to your `CLAUDE.md`.
3. Ask your agent something like: "Find my Codex conversation called X and
   pick up where it left off."

Needs `rg`, `jq`, and `zstd` (`brew install ripgrep jq zstd`). The Claude
Desktop paths are for macOS.
