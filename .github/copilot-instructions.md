# Copilot Instructions

This is a Hugo static site blog deployed to GitHub Pages at `https://anthonyabeo.github.io/`. The active theme is **PaperMod** (in `themes/PaperMod`). An older `hugo-coder` theme is also present as a git submodule but not in use.

## Common Commands

```bash
# Start local dev server with live reload
hugo server

# Build the site (output goes to public/)
hugo

# Create a new post
hugo new posts/my-post-title.md

# Create a new project page
hugo new projects/my-project.md
```

## Content Structure

- `content/posts/` — Blog posts (technical writing on systems topics)
- `content/projects/` — Project/paper writeups and summaries
- `static/images/` — Images organized into subdirectories by post topic (e.g., `static/images/unicode/`, `static/images/memory/`)
- `archetypes/default.md` — Template used when running `hugo new`

## Front Matter Conventions

Posts use YAML front matter. Common fields:

```yaml
---
title: "Post Title"
date: 2021-09-27T13:44:57Z
tags: ['tag1', 'tag2']
description: ""
slug: "url-slug"        # optional, overrides default URL slug
draft: true             # new content defaults to draft; set false to publish
---
```

Projects use a minimal front matter (title + date, no tags).

New content created via `hugo new` starts as `draft: true` — remember to set `draft: false` before publishing.

## Site Configuration

`config.yml` is the single site config file. Key settings:
- `theme: PaperMod` — active theme
- `buildDrafts: false` — drafts are excluded from production builds
- `menu.main` — controls the top navigation (currently only "Projects")
- `params.homeInfoParams` — the homepage bio text
- `params.socialIcons` — footer social links (Twitter, GitHub, LinkedIn)

To add a new menu item, append to `menu.main` in `config.yml`.
