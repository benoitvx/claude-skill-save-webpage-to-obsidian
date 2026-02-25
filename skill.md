---
name: save-webpage
description: >
  Saves a web article as a formatted markdown file into your Obsidian vault.
  Triggers: /save-webpage <url>, "save this article", "clip this page"
argument-hint: "<url>"
---

# Skill: Save Article to Obsidian

Extracts a web article and saves it as a clean markdown file with YAML frontmatter into your Obsidian vault.

## Configuration

Edit the path below to point to your Obsidian vault folder:

- **Output directory**: `~/path/to/your/obsidian-vault/Articles/`

## Output format

The markdown file follows this template:

```yaml
---
title: "Article title"
source: "https://full-url.com/article"
date: "YYYY-MM-DD"
summary: "AI-generated 2-sentence summary."
---
```

Followed by the article content in markdown (h1 title, paragraphs, lists, blockquotes, code blocks).

## Workflow

### Step 1 — Get the URL

The URL is passed as argument: `/save-webpage <url>`
If no URL is provided, ask the user.

### Step 2 — Extract the content

Use a **dual extraction strategy**: try Chrome browser automation first, fall back to WebFetch if Chrome is not connected.

#### Strategy A — Chrome (preferred)

1. Call `tabs_context_mcp` to get browser context
2. Create a new tab with `tabs_create_mcp`
3. Navigate to the URL with `navigate`
4. Wait 3 seconds with `computer` action `wait` to let the page load
5. Use `javascript_tool` to run the extraction script below

If `tabs_context_mcp` fails or Chrome is not connected, fall back to Strategy B.

#### Strategy B — WebFetch (fallback)

1. Use the `WebFetch` tool to fetch the URL
2. In the prompt, ask to extract: the article title, publication date (YYYY-MM-DD format), and the full article content as clean markdown (headings, paragraphs, lists, blockquotes, code blocks)
3. Parse the result to get title, date, and content

### JavaScript extraction script (for Strategy A)

```javascript
(() => {
  // Title
  const h1 = document.querySelector('article h1, main h1, .post-title, .entry-title, h1');
  const title = h1 ? h1.innerText.trim() : document.title.trim();

  // Publication date
  let date = '';
  const timeEl = document.querySelector('time[datetime]');
  if (timeEl) {
    date = timeEl.getAttribute('datetime').substring(0, 10);
  } else {
    const metaDate = document.querySelector(
      'meta[property="article:published_time"], meta[name="date"], meta[name="publication_date"], meta[name="DC.date"]'
    );
    if (metaDate) {
      date = metaDate.content.substring(0, 10);
    }
  }

  // Article content — target the main container
  const articleEl = document.querySelector('article, main, .post-content, .entry-content, .article-body, [role="main"]');
  const container = articleEl || document.body;

  const blocks = container.querySelectorAll('h1, h2, h3, h4, p, ul, ol, blockquote, pre, figure img');
  const lines = [];

  blocks.forEach(el => {
    const tag = el.tagName.toLowerCase();
    const text = el.innerText ? el.innerText.trim() : '';
    if (!text && tag !== 'img') return;

    if (tag === 'h1') lines.push(`# ${text}\n`);
    else if (tag === 'h2') lines.push(`## ${text}\n`);
    else if (tag === 'h3') lines.push(`### ${text}\n`);
    else if (tag === 'h4') lines.push(`#### ${text}\n`);
    else if (tag === 'p') lines.push(`${text}\n`);
    else if (tag === 'ul') {
      el.querySelectorAll(':scope > li').forEach(li => {
        lines.push(`- ${li.innerText.trim()}`);
      });
      lines.push('');
    }
    else if (tag === 'ol') {
      let i = 1;
      el.querySelectorAll(':scope > li').forEach(li => {
        lines.push(`${i}. ${li.innerText.trim()}`);
        i++;
      });
      lines.push('');
    }
    else if (tag === 'blockquote') lines.push(`> ${text}\n`);
    else if (tag === 'pre') lines.push(`\`\`\`\n${text}\n\`\`\`\n`);
    else if (tag === 'img') {
      const src = el.getAttribute('src');
      const alt = el.getAttribute('alt') || '';
      if (src) lines.push(`![${alt}](${src})\n`);
    }
  });

  JSON.stringify({ title, date, content: lines.join('\n') });
})()
```

### Step 3 — Generate the summary

From the extracted content, write a **2-sentence summary** in the **same language as the article**. The summary should be informative and concise, capturing the key points.

### Step 4 — Build the markdown file

Assemble the file with:
- YAML frontmatter (title, source, date, summary)
- The extracted markdown content

The filename is derived from the article title: replace `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|` with `-`, and truncate to 100 characters max. Extension: `.md`.

### Step 5 — Write the file

Use the `Write` tool to save the file to the configured output directory:

```
{output-directory}/{cleaned-filename}.md
```

### Step 6 — Confirm

Display to the user:
- The article title
- The generated summary
- The full path of the created file

## Error handling

- **Chrome not connected**: automatically fall back to WebFetch (Strategy B). If WebFetch also fails, inform the user.
- **JS-heavy / SPA page**: if Chrome extraction returns empty or very short content (< 100 characters), use `get_page_text` as a secondary fallback, then format the raw text as markdown.
- **Date not found**: leave `date` empty (`""`).
- **Title not found**: use `document.title` as fallback (Chrome) or the page `<title>` tag (WebFetch).
- **Write failed**: inform the user and offer to write to the Downloads folder instead.
