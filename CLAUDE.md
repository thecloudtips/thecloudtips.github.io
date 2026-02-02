# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based static blog using the Chirpy theme (v7.1+), hosted on GitHub Pages at thecloudtips.github.io with custom domain naluforge.com.

## Common Commands

### Development Server
```bash
./tools/run.sh                    # Start dev server at localhost:4000 with live reload
./tools/run.sh -H 0.0.0.0        # Bind to all interfaces (for container access)
./tools/run.sh -p                # Production mode
```

### Build and Test
```bash
./tools/test.sh                   # Production build + html-proofer validation
bundle exec jekyll b              # Build only (output to _site/)
```

### Dependencies
```bash
bundle install                    # Install Ruby gems
```

## Architecture

### Content Structure
- **`_posts/`** - Blog posts in Markdown with YAML frontmatter (title, date, categories, tags)
- **`_tabs/`** - Navigation pages (about, archives, categories, tags) with `order` property for sequencing
- **`_data/`** - Site data files (contact.yml for social links, share.yml for sharing config)

### Theme Architecture
The Chirpy theme is installed as a gem dependency. Theme files (layouts, includes, sass) come from the gem. To find theme source files:
```bash
bundle info --path jekyll-theme-chirpy
```

### Key Configuration
- **`_config.yml`** - Central configuration for site metadata, theme options, analytics, comments, and PWA settings
- **`_plugins/posts-lastmod-hook.rb`** - Auto-populates `last_modified_at` from git history for posts with multiple commits

### Static Assets
- **`assets/img/favicons/`** - Favicon files
- **`assets/img/personal/`** - Avatar and personal images
- **`assets/lib/`** - Git submodule pointing to chirpy-static-assets

## Deployment

GitHub Actions (`.github/workflows/pages-deploy.yml`) automatically builds and deploys on push to main. The workflow:
1. Builds with `JEKYLL_ENV=production`
2. Validates HTML links with html-proofer
3. Deploys to GitHub Pages

## Creating Content

### New Blog Post
Create file in `_posts/` with format `YYYY-MM-DD-title.md`:
```yaml
---
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS +/-TTTT
categories: [category1, category2]
tags: [tag1, tag2]
---
Content here...
```
