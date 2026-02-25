# save-article-to-obsidian

![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-blueviolet)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that saves any web article as a clean markdown file in your Obsidian vault. It extracts the article content, generates a short summary, and writes a properly formatted markdown file with YAML frontmatter.

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- Optionally: [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/imokhccebobalbpabakgmcckgpghkbkp) extension for better extraction from JS-heavy pages

## Installation

```bash
mkdir -p ~/.claude/skills/save-article
cp skill.md ~/.claude/skills/save-article/skill.md
```

Or clone this repo and copy:

```bash
git clone https://github.com/benoitvx/claude-skill-save-article-to-obsidian.git
cp claude-skill-save-article-to-obsidian/skill.md ~/.claude/skills/save-article/skill.md
```

## Configuration

Open `~/.claude/skills/save-article/skill.md` and edit the **Output directory** path in the `## Configuration` section to point to your Obsidian vault folder:

```markdown
- **Output directory**: `~/my-obsidian-vault/Articles/`
```

## Usage

```
/save-article https://example.com/some-article
```

Claude will:
1. Extract the article content (via Chrome or WebFetch)
2. Generate a 2-sentence summary in the article's language
3. Save a markdown file to your configured output directory

## Output format

Each saved article looks like this:

```yaml
---
title: "How Large Language Models Work"
source: "https://example.com/how-llms-work"
date: "2025-03-15"
summary: "This article explains the transformer architecture behind modern LLMs. It covers attention mechanisms, training processes, and recent scaling trends."
---
```

Followed by the full article content in clean markdown.

## Customization

You can edit `skill.md` to:

- **Change frontmatter keys** — rename `title`, `source`, `date`, `summary` to whatever your Obsidian setup expects
- **Force summary language** — replace "same language as the article" with a specific language (e.g. "in French")
- **Change the output template** — modify the YAML frontmatter structure or add extra fields (tags, categories, etc.)
- **Adjust the extraction script** — tweak CSS selectors for sites with non-standard layouts

## License

[MIT](LICENSE)
