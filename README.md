# diachkow.dev

Personal blog built with [Hugo](https://gohugo.io/) — no themes, no JavaScript, pure CSS terminal aesthetic.

Live at **[diachkow.dev](https://diachkow.dev)**

## Structure

```
├── hugo.toml                  # Hugo configuration
├── content/
│   ├── _index.md              # Homepage front matter
│   └── posts/                 # Blog posts (Markdown)
├── layouts/
│   ├── index.html             # Homepage template
│   └── _default/
│       └── single.html        # Post template
├── static/css/                # Stylesheets (Everblush color palette)
│   ├── variables.css
│   ├── base.css
│   ├── home.css
│   ├── single.css
│   └── syntax.css
└── .github/workflows/
    └── deploy.yml             # GitHub Pages deployment
```

## Running locally

Requires [Hugo extended](https://gohugo.io/installation/) (v0.156.0+).

```bash
# Dev server with live reload
hugo server

# Include drafts
hugo server -D

# Production build
hugo --minify
```

Site will be available at `http://localhost:1313/`.

## Deployment

Automatic via GitHub Actions on push to `main`. The workflow:

1. Builds with `hugo --minify --baseURL "https://diachkow.dev/"`
2. Deploys to GitHub Pages

DNS is managed through Cloudflare (A records + CNAME) pointing to GitHub Pages.

## Adding a new post

Create a new [page bundle](https://gohugo.io/content-management/page-bundles/) directory under `content/posts/`:

```
content/posts/my-new-post/
├── index.md           # Post content
├── screenshot.png     # Images used in the post
└── diagram.jpg
```

Edit `index.md` front matter:

```yaml
---
title: "My New Post"
date: 2026-03-01
tags: ["topic1", "topic2"]
description: "Short description for the post list."
draft: false
---

Post content in Markdown goes here.

![Screenshot](screenshot.png)
```

Images are referenced by filename since they live in the same directory.

Set `draft: true` to hide from production builds (visible with `hugo server -D`).

**Note:** After adding a post, manually add an entry to `layouts/index.html` to include it on the homepage.
