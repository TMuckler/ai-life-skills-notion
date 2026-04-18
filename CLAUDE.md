# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo distributes a single Claude Code skill: **`summarize`**. It contains no application code, no build system, and no tests. The two files that matter:

- `SKILL.md` — the skill definition loaded by Claude Code. Frontmatter (`name`, `description`, `user_invocable`) controls how the skill is discovered and invoked. The body is the operational prompt Claude follows when the skill fires.
- `README.md` — user-facing install/usage docs. Must stay in sync with `SKILL.md`.

Users install by copying `SKILL.md` to `~/.claude/skills/summarize/SKILL.md` and restarting Claude Code. There is nothing to compile or run.

## Editing the skill

When changing `SKILL.md`, preserve these load-bearing invariants — they are the skill's contract with end users:

- **Frontmatter triggers.** The `description` field is how Claude decides to invoke the skill. Keep the trigger phrases ("summarize this", "save this to Notion", "summarize this sermon", `/summarize`) — removing them silently breaks invocation.
- **Step ordering.** Step 0 (bootstrap: CLI tool check + Second Brain folder discovery/creation) must run before Step 1 on first use, and be skipped on subsequent runs. Sermon flow branches after Step 1b and bypasses Step 2.
- **Notion folder structure.** `Second Brain / {Article Summaries, Book Summaries, Youtube Summaries, Sermons}` with the exact emojis (🧠 📰 📚 🎬 ⛪). Renames break existing users' workspaces.
- **Model policy.** Summarization and sermon notes always use **Opus**. Never Haiku or Sonnet. This is called out explicitly multiple times in SKILL.md and is intentional.
- **Depth table (Step 2a).** The word-count-to-summary-length mapping is the skill's core quality lever. Book chapters get 300–600 words each; don't let edits collapse this into terse summaries.
- **Sermon prompt fidelity.** The sermon notes prompt (Step: "Sermon Notes Prompt") forbids fabrication. Any edit that loosens "based ONLY on what actually appears in the transcript" changes the skill's behavior in a way users rely on.
- **No databases/tags/wikilinks.** The skill is deliberately minimal (see README "Design choices"). Don't add Notion database creation, property tagging, or cross-page linking without an explicit request.
- **Page title rule.** Titles go in `properties.title` only, never in the `content` body — Notion renders titles itself and duplicates look broken.

## External dependencies referenced by the skill

The skill shells out to CLI tools at runtime on the end user's machine: `yt-dlp`, `defuddle`, `pdftotext` (poppler), `pandoc`, optionally `mlx_whisper` or ElevenLabs Scribe. It also requires the Notion MCP server to be connected. When editing extraction logic, keep the tool list in `SKILL.md` Step 0a, the Requirements table, and `README.md` Installation aligned.

## Validating changes

There is no automated test. To validate edits:

1. Copy the edited `SKILL.md` to `~/.claude/skills/summarize/SKILL.md`.
2. Restart Claude Code.
3. Ask "what skills do you have available?" to confirm it loaded.
4. Run the skill against representative inputs (a short article, a YouTube video, a sermon URL) and inspect the resulting Notion pages.
