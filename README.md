# fasih2611.github.io

Personal site, built with Jekyll and hosted on GitHub Pages at
<https://fasih2611.github.io>.

## Writing a post

Create `_posts/YYYY-MM-DD-slug.md`:

```markdown
---
title: "Post title"
date: 2026-08-10
description: "Optional one-liner for the blog index."
tags: [optional, tags]
---

Content in markdown.
```

Then commit and push — GitHub rebuilds the site in about a minute.

## Editing the rest

| What | Where |
|---|---|
| Name, email, links, site title | `_config.yml` |
| About page | `index.md` |
| Projects list | `projects.md` |
| Styling | `assets/css/style.css` |
| Page shell / nav | `_layouts/default.html` |

## Local preview (optional)

Requires Ruby. Not needed to publish — GitHub builds the site server-side.

```bash
gem install bundler jekyll
bundle exec jekyll serve
```
