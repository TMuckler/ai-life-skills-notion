# summarize

A Claude Code skill that turns any URL, file, or pasted text into a clean, well-structured summary page inside your Notion workspace.

## What it does

Point it at a YouTube video, web article, PDF, EPUB book, podcast episode, or sermon. It extracts the content, generates a proportional summary using Claude Opus, and pushes a formatted page into your Notion workspace under a simple folder structure.

No tags. No databases. No wikilinks. No reference pages. Just the summary, in the right folder.

## Notion folder structure

The skill creates (or uses) this structure on first run:

```
Second Brain/
├── 📰 Article Summaries
├── 📚 Book Summaries
├── 🎬 Youtube Summaries
└── ⛪ Sermons
```

Each summary lands in the folder matching its content type.

## Features

- **Universal content extraction** — YouTube (with auto-subs or whisper transcription), web articles (via defuddle), PDFs (via pdftotext), EPUBs (via pandoc), podcasts, pasted text
- **Proportional depth** — a 5-minute video gets a short summary, a 600-page book gets a chapter-by-chapter breakdown
- **Parallel Opus subagents** for long content so books and multi-hour videos don't take forever
- **Dedicated sermon flow** — triggered by "summarize this sermon" with a purpose-built teaching-notes prompt that captures structure, illustrations, theological foundations, and preacher's voice without fabricating content
- **Time trimming for sermons** — skip the worship and announcements with "starting at 47:00" or "from 47:00 to 1:32:00"
- **Audience adaptation** — "summarize for a high school student" or "summarize for an expert"
- **Clean Notion formatting** — metadata quote block at the top, TLDR callout, proportional sections, no frontmatter clutter

## Installation

**1. Install CLI dependencies** (required for content extraction):

```bash
brew install yt-dlp poppler pandoc
npm install -g defuddle
```

Optional for local audio transcription:

```bash
pip install mlx-whisper
```

Or set `ELEVENLABS_API_KEY` to use ElevenLabs Scribe instead of local transcription.

**2. Install the skill:**

```bash
mkdir -p ~/.claude/skills/summarize
# Copy SKILL.md into ~/.claude/skills/summarize/
```

**3. Connect Notion to Claude Code** via MCP if you haven't already. See [Notion's MCP setup docs](https://developers.notion.com/docs/mcp).

**4. Restart Claude Code** and verify the skill loaded:

```
what skills do you have available?
```

You should see `summarize` listed.

## Usage

Just give Claude a URL, a file, or some pasted text along with intent to summarize:

```
summarize https://example.com/some-article
```

```
/summarize https://youtube.com/watch?v=xxxxx
```

```
summarize this sermon starting at 12:30 — https://youtube.com/watch?v=xxxxx
```

```
summarize this PDF for my second brain: /path/to/paper.pdf
```

Optional controls:

- Audience: "summarize for a beginner" / "for an expert" / "for a high school student"
- Depth: "tldr only" / "deep dive"

## First run

On first run, the skill will:

1. Check that `yt-dlp`, `defuddle`, `pdftotext`, and `pandoc` are installed (prompting to install any that are missing)
2. Search your Notion for a page titled "Second Brain" — if not found, create it at the workspace level along with the four folder pages

Subsequent runs skip the bootstrap.

## Design choices

This skill is intentionally minimal compared to a "knowledge graph" style workflow:

- **No tags or properties** — folders alone handle categorization
- **No reference/people/daily pages** — summaries stand on their own
- **No transcripts saved** — they're used for extraction, then discarded
- **Bold key terms inline** instead of linking them

If you want a link-heavy, interconnected second brain with reference pages for every concept, this isn't the skill for you. If you want fast, clean summaries that are easy to skim later, this is it.

## Credits

Adapted from [reysu/ai-life-skills](https://github.com/reysu/ai-life-skills). Rewritten for Notion with the complexity stripped out.
