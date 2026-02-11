# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Multilingual (English/Chinese) technical blog built with [Zola](https://www.getzola.org/) static site generator using the `slim` theme.

## Branch Model

- **`code`**: Source branch (all edits happen here)
- **`master`**: Deployment branch (auto-published by GitHub Actions, never edit directly)

## Build Commands

```bash
# Build the site (outputs to public/)
zola build

# Local dev server with live reload
zola serve

# Check for broken links and other issues
zola check
```

Zola version: **0.22.1** (pinned in `.github/workflows/deploy.yml`).

## Deployment

Push to `code` triggers GitHub Actions (`.github/workflows/deploy.yml`) which runs `zola build` and publishes `public/` to `master` via GitHub Pages.

## Content Authoring

- Articles live in `content/` as Markdown with TOML frontmatter (`+++` delimiters)
- Multilingual pairs: `article.md` (English) + `article.zh.md` (Chinese)
- Co-located pages (e.g., `content/raft/`) share assets between language versions
- Use `<!-- more -->` to mark the summary/excerpt break

## Architecture

### Multilingual Routing

- Config: `default_language = "en"` with `[languages.zh]` in `config.toml`
- Chinese pages get `/zh/` path prefix automatically
- Chinese search index is disabled (`build_search_index = false`) because Zola's elasticlunr doesn't support Chinese

### Templates

- **Theme templates**: `themes/slim/templates/` — base layout, page/section rendering, macros (header, footer, lang_toggle, pagination)
- **Custom shortcodes**: `templates/shortcodes/` — `raft_table.html` (multi-column image table), `resize_image.html` (image processing wrapper)

### Key Tera Template Patterns

Shortcodes that reference co-located assets must strip the `/zh/` prefix from `page.path` so both language versions find the same files:

```tera
{% set base_path = page.path | replace(from="/zh/", to="/") %}
```

Tera cannot chain a filter with string concatenation (`page.path | replace(...) ~ suffix` fails). Always assign to an intermediate variable first.

### Config Structure (config.toml)

TOML ordering matters: top-level `taxonomies` array must appear **before** the `[languages.zh]` section, otherwise Zola misparses it.
