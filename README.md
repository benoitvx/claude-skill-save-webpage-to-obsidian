# save-webpage-to-obsidian

![Claude Code Skill](https://img.shields.io/badge/Claude_Code-Skill-blueviolet)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that saves any web article as a clean markdown file in your Obsidian vault. It extracts the article content via WebFetch, generates a short summary, and writes a properly formatted markdown file with YAML frontmatter.

## Features

- **WebFetch extraction** — fast, reliable, no browser needed
- **Chrome fallback** — automatically falls back to Chrome if WebFetch fails (JS-heavy pages, blocked content)
- **Duplicate detection** — checks existing files before saving to avoid duplicates
- **AI summary** — generates a 2-sentence summary for each article

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- Optionally: [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/imokhccebobalbpabakgmcckgpghkbkp) extension as fallback for JS-heavy pages

## Installation

```bash
mkdir -p ~/.claude/skills/save-webpage
cp skill.md ~/.claude/skills/save-webpage/skill.md
```

Or clone this repo and copy:

```bash
git clone https://github.com/benoitvx/claude-skill-save-webpage-to-obsidian.git
cp claude-skill-save-webpage-to-obsidian/skill.md ~/.claude/skills/save-webpage/skill.md
```

## Configuration

Open `~/.claude/skills/save-webpage/skill.md` and edit the paths in the `## Chemins` section to point to your Obsidian vault:

```markdown
- **Vault Obsidian** : `~/my-obsidian-vault`
- **Dossier cible** : `Articles/` (subfolder for saved articles)
- **Dossier téléchargements** : `~/Downloads` (fallback if write fails)
```

## Usage

```
/save-webpage https://example.com/some-article
```

Claude will:
1. Check for duplicates in existing saved articles
2. Extract the article content (via WebFetch, Chrome as fallback)
3. Generate a 2-sentence summary
4. Save a markdown file to your configured output directory

## Output format

Each saved article looks like this:

```yaml
---
titre: "How Large Language Models Work"
source: "https://example.com/how-llms-work"
date_publication: "2025-03-15"
résumé: "This article explains the transformer architecture behind modern LLMs. It covers attention mechanisms, training processes, and recent scaling trends."
---
```

Followed by the full article content in clean markdown.

## Customization

You can edit `skill.md` to:

- **Change frontmatter keys** — rename `titre`, `source`, `date_publication`, `résumé` to whatever your Obsidian setup expects
- **Change summary language** — replace "en français" with any language
- **Change the output template** — modify the YAML frontmatter structure or add extra fields (tags, categories, etc.)
- **Adjust the WebFetch prompt** — tweak the extraction prompt for better results on specific types of pages

## License

[MIT](LICENSE)
