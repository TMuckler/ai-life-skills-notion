---
name: summarize
description: Summarize any content (YouTube video, article, whitepaper/PDF, podcast episode, book chapter, etc.) into a rich Notion page with section-by-section breakdowns, organized into the Second Brain folder structure. Use when the user provides a URL, file, or content to summarize and wants it saved to Notion. Triggers on phrases like "summarize this", "save this to Notion", "add to my second brain", "/summarize", or any URL/file provided with intent to summarize and store. Also triggers for sermon summarization when the user says "summarize this sermon."
user_invocable: true
---

# Summarize to Notion

Universal content summarizer. Takes any input (YouTube video, web article, whitepaper/PDF, epub book, podcast, lecture, sermon) and produces a rich, well-formatted Notion page inside the user's **Second Brain** folder structure.

## Notion Folder Structure

All output goes under a top-level page called **Second Brain** in the user's Notion workspace. Inside it:

| Folder (Notion page) | Content type |
|---|---|
| `Article Summaries` | Web articles, blog posts, pasted text |
| `Book Summaries` | Books (EPUB), PDFs, whitepapers |
| `Youtube Summaries / Channel Name` | YouTube video summaries, organized by channel |
| `Sermons` | Sermon notes (from YouTube, audio, or PDF) |

YouTube summaries use a two-level structure: a channel sub-page lives inside `Youtube Summaries`, and individual summary pages live inside that channel sub-page. The channel name comes from the `%(channel)s` field returned by `yt-dlp`.

## Requirements

**CLI tools** -- install these before first use:

| Tool | Purpose | Install |
|---|---|---|
| `yt-dlp` | YouTube/podcast download + metadata + subs | `brew install yt-dlp` or `pip install yt-dlp` |
| `defuddle` | Web article extraction | `npm install -g defuddle` |
| `pdftotext` | PDF text extraction | `brew install poppler` |
| `pandoc` | EPUB / DOCX to markdown | `brew install pandoc` |
| `mlx_whisper` (optional) | Local audio transcription fallback | `pip install mlx-whisper` |

Alternative to `mlx_whisper`: set `ELEVENLABS_API_KEY` to use ElevenLabs Scribe for transcription.

## Step 0: Bootstrap check (first run only)

Before doing any work, verify the environment is ready. **Skip any check that already passes.** Do not re-run Step 0 on subsequent invocations if the initial setup succeeded.

### 0a. Check required CLI tools

```bash
for tool in yt-dlp defuddle pdftotext pandoc; do
  command -v "$tool" >/dev/null 2>&1 || echo "MISSING: $tool"
done
```

For each missing tool, tell the user what's missing and **ask before installing**. If the user declines, note which content types will fail.

### 0b. Find or create the Second Brain folder structure in Notion

Search Notion for a page called "Second Brain":

```
Notion:notion-search
  query: "Second Brain"
  filters: {}
  max_highlight_length: 0
  page_size: 5
```

**Important:** The search is semantic, not exact match. Verify the returned result's title is exactly "Second Brain" before using it. If no exact match is found, create the structure.

If not found, create the structure. Omit the `parent` field to create at the workspace root (the tool creates private workspace-level pages when no parent is given):

```
Notion:notion-create-pages
  pages: [{ properties: { title: "Second Brain" }, icon: "🧠" }]
```

If this fails with a validation error about a missing `parent`, ask the user: "Where in your Notion workspace should I create the Second Brain page? Give me a link or name of a parent page."

Then create the four child pages under the returned Second Brain page_id:

```
Notion:notion-create-pages
  parent: { page_id: "<second_brain_page_id>" }
  pages: [
    { properties: { title: "Article Summaries" }, icon: "📰" },
    { properties: { title: "Book Summaries" }, icon: "📚" },
    { properties: { title: "Youtube Summaries" }, icon: "🎬" },
    { properties: { title: "Sermons" }, icon: "⛪" }
  ]
```

Cache the page IDs for the four folders for use in later steps. Channel sub-pages inside `Youtube Summaries` are created on demand in Step 3 — do not pre-create them here.

If the Second Brain page exists, fetch it with `Notion:notion-fetch` using its page ID to discover child pages. If any of the four folders are missing, create only the missing ones.

Once Step 0 passes, proceed to Step 1.

## Step 1: Detect content type and extract text

Before extracting any content, ensure the working directory exists:

```bash
mkdir -p /tmp/summarize
```

### YouTube video
```bash
# Get metadata
yt-dlp --cookies-from-browser chrome \
  --print "%(id)s|%(title)s|%(duration)s|%(upload_date)s|%(view_count)s|%(channel)s|%(channel_id)s" \
  --no-download "<URL>"

# Try auto-subtitles first (fastest, free)
yt-dlp --cookies-from-browser chrome \
  --write-auto-sub --sub-lang en --sub-format json3 \
  --skip-download -o "/tmp/summarize/%(id)s" "<URL>"
```

If auto-subs exist, extract text from the JSON3 file. If not, or if quality is poor:
- Download audio and transcribe (ask user: local mlx_whisper or ElevenLabs Scribe)

### Web article / blog post
```bash
defuddle parse "<URL>" --md -o /tmp/summarize/article.md
```

Extract title, author, date, domain from defuddle metadata:
```bash
defuddle parse "<URL>" -p title
defuddle parse "<URL>" -p domain
```

### PDF
```bash
pdftotext "<path>" /tmp/summarize/paper.txt
```

### EPUB (books)
```bash
pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/book.md

# Extract chapter boundaries from TOC:
pandoc "<path>" -t json | python3 -c "
import json, sys
doc = json.load(sys.stdin)
for block in doc['blocks']:
    if block['t'] == 'Header':
        level = block['c'][0]
        text = ''.join(
            item['c'] if item['t'] == 'Str' else ' ' if item['t'] == 'Space' else ''
            for item in block['c'][2]
        )
        print(f'L{level}: {text}')
"
```

**Chapter splitting strategy for books:**
1. Extract full text with `pandoc`
2. Identify chapter boundaries from headers
3. Split into one chunk per chapter
4. Dispatch parallel Opus subagents, one per chapter
5. A typical book produces chapters of ~3-5k words each

**For very long books (>30 chapters):** batch chapters into groups of ~5 per subagent.

### Other files (txt, docx, etc.)

For `.docx`: `pandoc "<path>" -t markdown --wrap=none -o /tmp/summarize/doc.md`

For plain text: read directly.

### Pasted text
Read directly from user message.

### Podcast episode
If the podcast URL is a YouTube link, follow the YouTube extraction path above. For non-YouTube podcast URLs (e.g. Apple Podcasts, Spotify, RSS feed links):

1. Try `yt-dlp` first -- it supports many podcast platforms beyond YouTube:
```bash
yt-dlp --cookies-from-browser chrome \
  --write-auto-sub --sub-lang en --sub-format json3 \
  --skip-download -o "/tmp/summarize/podcast" "<URL>"
```

2. If `yt-dlp` can't extract subtitles, download the audio and transcribe:
```bash
yt-dlp --cookies-from-browser chrome \
  -x --audio-format mp3 \
  -o "/tmp/summarize/podcast.%(ext)s" "<URL>"
```
Then transcribe via mlx_whisper or ElevenLabs Scribe (ask user which).

3. If `yt-dlp` doesn't support the URL at all, ask the user to provide either a direct audio file or a transcript.

## Step 1b: Sermon detection and transcript trimming

**Trigger:** The user says "summarize this sermon" (or similar). This keyword distinguishes a sermon from a regular YouTube video. Without it, YouTube URLs follow the normal YouTube summary flow.

### Time trimming

Sermon videos often contain worship, announcements, etc. before the preaching. The user may provide:

- **Start time only:** "summarize this sermon starting at 47:00"
- **Start and end time:** "summarize this sermon from 47:00 to 1:32:00"
- **No time given:** Use the full transcript

When a start/end time is provided, trim the transcript to that window before processing:

```bash
python3 -c "
import json, sys

start = int(sys.argv[1])
end = int(sys.argv[2]) if len(sys.argv) > 2 else float('inf')

with open(sys.argv[3]) as f:
    subs = json.load(f)

for event in subs.get('events', []):
    t = event.get('tStartMs', 0) / 1000
    if start <= t <= end:
        segs = event.get('segs', [])
        text = ''.join(s.get('utf8', '') for s in segs).strip()
        if text:
            print(text)
" <start_seconds> <end_seconds> /tmp/summarize/<id>.en.json3
```

### Sermon processing flow

When content is identified as a sermon, the skill **bypasses Step 2** (depth planning and section dispatch) and uses the dedicated sermon notes generator instead of the normal summary assembly:

1. Step 0 (bootstrap)
2. Step 1 (extract transcript)
3. Step 1b (this step, trim if needed)
4. **Sermon notes generation** (see sermon prompt below)
5. Step 3, page creation only (push sermon notes to Notion using the sermon page structure)

## Step 2: Determine depth and plan sections

Read the full extracted text. Identify the natural sections/chapters/topics.

### 2a. Determine summary depth from source length

Summary length must be **proportional** to the source material:

| Source word count | Source examples | Target summary words | Sections | TLDR |
|---|---|---|---|---|
| <1,500 | 5-min video, short article | 200-400 | 1-2 | 2 sentences |
| 1,500-5,000 | 10-20 min video, blog post | 500-1,200 | 3-5 | 3 sentences |
| 5,000-15,000 | 30-60 min video, long article | 1,500-3,000 | 5-8 | 3-4 sentences |
| 15,000-40,000 | 1-3 hr video/podcast, long paper | 3,000-6,000 | 8-15 | 4-5 sentences |
| 40,000-80,000 | Short book, multi-hour series | 5,000-10,000 | 15-25 | 5 sentences |
| 80,000+ | Full book (200+ pages) | 8,000-15,000 | 20-40 | 5 sentences |

**The ratio is roughly 1:5 to 1:10.** Denser/more technical content skews higher; conversational/repetitive content skews lower.

**For videos/podcasts**, estimate source words from duration: ~150 words/minute conversational, ~120 for interviews, ~170 for scripted.

**Per-section depth**: each section's word budget should be proportional to its share of the source material.

### 2b. Plan sections and dispatch

**For long content (>3000 source words):** dispatch parallel **Opus** subagents, one per section, to summarize simultaneously. Each subagent gets:
- The section text
- The audience level
- A specific word count target (from 2a)
- Instructions to format output in Notion-compatible markdown

**For short content (<3000 source words):** summarize directly without subagents.

**NEVER use Haiku or Sonnet for summarization.** Always Opus.

**CRITICAL -- Book summary depth requirement:**
- Each chapter MUST get its own `## Chapter N: Title` section with a **substantial** summary (300-600 words per chapter)
- Do NOT batch multiple chapters into a single brief paragraph
- Include key arguments, data points, examples, and quotes from each chapter
- A 10-chapter book should produce ~3000-6000 words of summary
- A 30-chapter book should produce ~5000-10000 words

## Step 3: Assemble and push to Notion

### Content type routing

| Content type | Notion parent page | Icon |
|---|---|---|
| YouTube video | Youtube Summaries / Channel Name | 🎬 |
| Article / blog | Article Summaries | 📰 |
| Whitepaper / PDF | Book Summaries | 📄 |
| EPUB / book | Book Summaries | 📚 |
| Podcast episode | Article Summaries | 🎙️ |
| Lecture / talk | Article Summaries | 🎓 |
| Sermon | Sermons / Preacher or Church | ⛪ |

### YouTube channel sub-page lookup (YouTube only)

The channel name comes from `%(channel)s` in the `yt-dlp` metadata output. Before creating the summary page, check whether a sub-page for that channel already exists under `Youtube Summaries`:

```
Notion:notion-search
  query: "<channel name>"
  filters: {}
  page_url: "<youtube_summaries_page_id>"
  max_highlight_length: 0
  page_size: 5
```

Verify the returned result's title matches the channel name exactly. If found, use that sub-page as the parent. If not found, create the channel sub-page first:

```
Notion:notion-create-pages
  parent: { page_id: "<youtube_summaries_page_id>" }
  pages: [{ properties: { title: "<Channel Name>" }, icon: "📺" }]
```

Cache the channel sub-page ID. Then create the summary page under it.

### Page structure (non-sermon)

The Notion page content should be structured as:

```markdown
> **Source:** [Title or URL](url)
> **Author/Creator:** Name
> **Date:** YYYY-MM-DD
> **Duration:** (if video/audio)
> **Topics:** Topic1, Topic2, Topic3

---

> 💡 **TLDR**
> [Overview sentences per depth table. What is it about, who made it, key takeaways?]

## [Section 1 Title]

[Summary paragraphs]

## [Section 2 Title]

[...]

## See Also

- [Related Page Title](https://www.notion.so/<page_id>)
```

**Topics line:** After summarizing, identify 2-5 topic keywords that describe the content's main theological, practical, or subject-matter themes. Use consistent vocabulary across summaries so Notion search can surface related pages (e.g., always "Sovereignty of God" not sometimes "God's Sovereignty"). Good topic examples: Soteriology, Prayer, Evangelism, Marriage, Idolatry, Holy Spirit, Assurance, Forgiveness, Ecclesiology, Hermeneutics, Church History, Apologetics, Sanctification, Eschatology, Covenant Theology, Spiritual Disciplines.

### Page structure (book)

```markdown
> **Author:** Name
> **Published:** Year
> **ISBN:** (if known)
> **Topics:** Topic1, Topic2, Topic3

---

> 💡 **TLDR**
> [Overview of the book's thesis and contribution]

## Chapter 1: [Title]

[Substantial chapter summary]

## Chapter 2: [Title]

[...]

## See Also

- [Related Page Title](https://www.notion.so/<page_id>)
```

### Formatting rules for Notion

**Before assembling any content**, try fetching the Notion enhanced markdown spec for the most current syntax:

```
Notion:notion-fetch
  id: "notion://docs/enhanced-markdown-spec"
```

If this succeeds, follow the spec. If it fails (this URI is not always available), use the safe defaults below.

**Critical:** Do NOT include the page title in the content body. The title goes in `properties: { title: "..." }` only. Notion displays it automatically.

Safe markdown that always works in Notion `content`:

1. `> 💡 **TLDR**` for the overview block (renders as a blockquote with emoji)
2. `> 💬` for notable quotes (with speaker attribution)
3. **Bold** key terms and important concepts inline. **Bold + link** Scripture references to BibleGateway (see Bible verse linking section below)
4. `---` horizontal rules to separate metadata from content
5. Headings: `##` for major sections, `###` for subsections
6. No frontmatter/YAML -- Notion doesn't use it
7. Metadata goes in a quote block at the top of the page, not in properties
8. Keep formatting clean: paragraphs, headers, bold, italic, quotes, code blocks, bullet lists where appropriate
9. Toggle blocks use HTML-style `<details><summary>Title</summary>Content</details>` syntax
10. Heading colors use `{color="blue"}` syntax after the heading text (e.g. `## Section {color="blue"}`)

### Creating the page

Use `Notion:notion-create-pages` to create the summary page:

```
Notion:notion-create-pages
  parent: { page_id: "<appropriate_folder_page_id>" }
  pages: [{
    properties: { title: "<Content Title>" },
    icon: "<emoji per content type table>",
    content: "<assembled markdown content>"
  }]
```

### Bible verse linking

Every Scripture reference in the summary body should be a clickable link to BibleGateway (ESV). Use this URL pattern:

```
https://www.biblegateway.com/passage/?search=<encoded_reference>&version=ESV
```

Format the reference as a markdown link with the text bolded:

```markdown
**[Romans 8:28](https://www.biblegateway.com/passage/?search=Romans+8%3A28&version=ESV)**
```

URL encoding rules:
- Spaces become `+`
- Colons become `%3A`
- Dashes (verse ranges) become `-` (no encoding needed)
- Examples:
  - `Romans 8:28-30` → `Romans+8%3A28-30`
  - `1 Corinthians 2:14` → `1+Corinthians+2%3A14`
  - `Psalm 139:14` → `Psalm+139%3A14`

Apply this to all Scripture references in the summary body, the TLDR, and the metadata `> **Scripture:**` line (for sermons). Do NOT apply it inside `> 💬` quote blocks — those should preserve the speaker's words without turning them into links.

### See Also cross-links

After the final section of every summary (before any closing horizontal rule), add a `## See Also` section that links to 1-5 related pages already in the Second Brain. To find related pages, search the user's Notion workspace using the topic keywords from the `> **Topics:**` line:

```
Notion:notion-search
  query: "<topic keyword>"
  filters: {}
  page_url: "<second_brain_page_id>"
  max_highlight_length: 0
  page_size: 5
```

Run 2-3 searches using different topic keywords. Collect unique results, exclude the page being created, and verify each result is actually relevant (not just a semantic false positive). Format as:

```markdown
## See Also

- [Page Title](https://www.notion.so/<page_id_no_dashes>)
- [Page Title](https://www.notion.so/<page_id_no_dashes>)
```

If the Second Brain is empty or no related pages exist, omit the See Also section entirely. Do not force it.

**Large content warning:** For very long summaries (books with 20+ chapters, 8,000+ words of summary), the content may exceed size limits in a single call. If page creation fails due to content size, split the workflow:
1. Create the page with the metadata block + TLDR + first batch of sections
2. Fetch the created page to get the current content
3. Use `Notion:notion-update-page` with `command: "update_content"` and a `content_updates` entry where `old_str` matches the last section already on the page and `new_str` includes that section plus the remaining sections appended after it

### Audience adaptation

- **High school / college student**: plain language, analogies, explain jargon inline
- **General reader**: balanced, explain key terms but don't over-simplify
- **Expert**: technical language fine, focus on novel contributions and critiques

## Sermon Notes Prompt

When content is identified as a sermon, pass the full (or trimmed) transcript to an **Opus** subagent with this system prompt. The subagent's output becomes the body of the Notion page.

---

Read the full transcript below. Create a structured, fully comprehensive set of teaching notes that accurately captures everything delivered in the sermon.

**CRITICAL INSTRUCTION:** Create notes based ONLY on what actually appears in the transcript. Do not fabricate, imagine, or infer content that isn't explicitly present. If the preacher references something without explaining it, note that it was mentioned but not elaborated. Accuracy to the source material is paramount.

**FORMAT GUIDELINES:** Use a balanced mix of prose and bullet points:
- Write in prose for: theological explanations, doctrinal frameworks, exposition, extended illustrations/testimonies, connecting biblical themes, application development, and reflective insights
- Use bullets for: lists of principles, characteristics, action steps, structured breakdowns, key concepts, diagnostic questions, and specific examples

**REQUIRED STRUCTURE** (adapt as needed based on actual content):

1. **Sermon Overview** -- Brief summary of the text, main theme, and sermon focus
2. **Text and Context** -- The biblical passage(s) preached, relevant background, and textual observations
3. **Core Message** -- The main teaching content, organized by major points or movements
4. **Supporting Evidence and Illustrations** -- Biblical examples, cross-references, personal testimonies, contemporary illustrations, and case studies
5. **Theological Foundations** -- Doctrinal principles and theological framework underlying the message
6. **Practical Applications** -- How to apply the teaching in daily Christian life
7. **Pastoral Exhortations** -- Direct challenges, encouragements, warnings, or calls to action
8. **Gospel Connections** -- How the message points to Christ and the gospel (if applicable)
9. **Key Takeaways** -- Main principles, memorable quotes, and central truths
10. **Reflective Questions** -- Questions for personal meditation or small group discussion (if provided)
11. **Call to Response** -- Any altar calls, commitments encouraged, or next steps presented

**CONTENT REQUIREMENTS:** Capture with precision:
- The preacher's exact phrasing for key statements, theological propositions, and memorable declarations
- Complete details of any personal testimonies or illustrations (the situation, the tension, the resolution, the spiritual lesson)
- All biblical passages referenced, quoted, or exposited
- Cross-references and supporting scriptures cited
- Any historical, cultural, or linguistic context provided for the text
- The logical flow and argument structure of the sermon
- Transitions between major points and how ideas build on each other
- Repeated phrases or themes that signal emphasis
- Any specific statistics, quotes from other sources, or research mentioned

**CONTEXT MARKERS:** Include when:
- The preacher references previous sermons or ongoing series themes
- Congregational interaction occurs (responses, amens, participation)
- The preacher signals a shift in the message ("Now watch this...", "Here's the heart of it...")
- Something is alluded to but not fully developed
- The preacher explicitly marks something as the main point

**MAINTAIN THE PREACHER'S VOICE:**
- Use direct quotes for key declarations and powerful statements
- Preserve the cadence and style of testimonies and illustrations
- Keep rhetorical questions and call-and-response patterns
- Note when the preacher uses repetition for emphasis
- Capture the tone (exhortational, pastoral, prophetic, teaching, celebratory)

**WHAT TO AVOID:**
- Do not invent theological content not in the sermon
- Do not over-interpret casual comments as major doctrinal points
- Do not create systematic frameworks the preacher didn't present
- Do not add application points not actually given
- Do not include generic Christian advice not taught in this specific message
- Do not sanitize or overly polish raw, powerful moments of preaching
- Do not add gospel presentations if the preacher didn't include one

---

### Sermon page structure

```markdown
> **Preacher:** Name
> **Church:** Church Name
> **Series:** Series Name (if applicable)
> **Scripture:** Book Chapter:Verse-Verse
> **Date:** YYYY-MM-DD
> **Source:** [Watch](YouTube URL)
> **Topics:** Topic1, Topic2, Topic3

---

> 💡 **TLDR**
> [Brief overview: preacher, text, main thrust of the sermon]

[Sermon notes content generated by the prompt above]
```

**Sermon page location:** Under the **Sermons** folder page.

If the preacher or church can be determined, check if a sub-page for that preacher/church already exists under Sermons:

```
Notion:notion-search
  query: "<Preacher or Church Name>"
  filters: {}
  page_url: "<sermons_folder_page_id>"
  max_highlight_length: 0
  page_size: 5
```

Verify the returned result's title matches exactly. If found, create the sermon note under that sub-page. If not found, create a new sub-page under Sermons for the preacher/church, then create the sermon note under it.

## Inputs

- **Source**: URL, file path, or pasted text
- **Audience** (optional): defaults to "general reader." User may specify (e.g. "high school student", "expert", "5-year-old")
- **Depth** (optional): defaults to "full." User may request "tldr only", "section-by-section", or "deep dive"

## Model usage

| Task | Model |
|------|-------|
| Content extraction | Scripts (defuddle, pdftotext, yt-dlp) |
| Section summarization | **Opus** subagents (parallel) |
| Sermon notes generation | **Opus** (single subagent, full transcript) |
| **NEVER** | **Haiku or Sonnet** |

## Key rules

1. **No databases, no tags, no references, no people pages, no daily notes** -- just the summary page in the right folder. YouTube summaries go in `Youtube Summaries / Channel Name / summary`, not directly in `Youtube Summaries`.
2. **Bold key terms** inline instead of linking them
3. **Metadata in a quote block** at the top of the page, not in frontmatter or properties
4. **`> 💡 **TLDR**`** is mandatory on every summary
5. **Parallel Opus subagents** for long content
6. **Audience-appropriate language** -- match the user's requested level
7. **Always include the source** -- URL, embed, or attribution in the metadata block
8. **Proportional depth** -- longer source = longer summary, per the depth table
9. **Sermon flow is separate** -- sermons bypass the normal summary steps and use the dedicated sermon notes prompt
10. **No transcripts saved** -- transcripts are used for processing only, not stored as separate pages
11. **Topics line is mandatory** -- every summary gets a `> **Topics:**` line with 2-5 consistent keywords in the metadata block
12. **Bible verses link to BibleGateway** -- every Scripture reference in the summary body becomes a bold clickable link to the ESV text on BibleGateway. Do not link verses inside quote blocks.
13. **See Also cross-links** -- after the final section, search the Second Brain for related pages and add a `## See Also` section linking to 1-5 matches. Omit if none found.
