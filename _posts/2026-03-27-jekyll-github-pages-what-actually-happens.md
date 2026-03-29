---
layout: default
title: "Jekyll and GitHub Pages: What Actually Happens When You Push"
date: 2026-03-27
---

## Jekyll and GitHub Pages: What Actually Happens When You Push

Most developers know that GitHub Pages turns a repository into a website. Fewer know exactly what happens between `git push` and a rendered HTML page showing up at `username.github.io`. The answer is Jekyll, and the mechanism is simpler than most people assume.

### The short version

GitHub Pages is a static site host. Jekyll is a static site generator. When you push Markdown files to a GitHub Pages repo, GitHub runs Jekyll on them automatically, converts everything to HTML, and serves the result. No server, no runtime, no database. Just files.

### What Jekyll actually does

Jekyll is a Ruby program that does one thing: it reads a directory of files, applies templates to them, and outputs a folder of static HTML. The input is Markdown (plus some YAML configuration). The output is a complete website.

Here is the transformation in pseudocode:

```text
for each .md file in the repo:
    1. read the YAML front matter (title, date, layout)
    2. convert the Markdown body to HTML
    3. inject that HTML into the layout template
    4. write the result to _site/<path>.html

copy all static assets (CSS, images) to _site/
```

That is it. The `_site/` folder is your website. GitHub Pages serves it over HTTPS from their CDN.

### The role of `_config.yml`

This file is Jekyll's single configuration file. It controls:

- **theme**: which CSS and layout templates to use
- **title/description**: metadata injected into the HTML `<head>`
- **permalink**: URL structure for blog posts
- **plugins**: additional processing steps (e.g., `jekyll-remote-theme` loads themes from GitHub repos)

A minimal example:

```yaml
remote_theme: pages-themes/minimal@v0.2.0
plugins:
  - jekyll-remote-theme
title: My Site
permalink: /blog/:year/:month/:title/
```

This is enough to turn a repo of Markdown files into a themed website with a blog.

### Front matter: the YAML block

Every Markdown file that Jekyll should process needs a YAML block at the top:

```yaml
---
layout: default
title: "My Post Title"
date: 2026-03-28
---
```

Jekyll uses this to decide which template to apply, what the page title is, and how to sort blog posts by date. Files without front matter are copied as-is (useful for images, CSS, etc.).

### Why this architecture works

The design is opinionated in ways that make it hard to misuse:

**No build step on your machine.** You push Markdown. GitHub runs Jekyll. You never install Ruby, never run `bundle exec jekyll build`, never debug Gemfile conflicts. GitHub handles all of it.

**No runtime dependencies.** The output is plain HTML and CSS. There is no JavaScript framework, no server-side rendering, no API layer. The site loads fast because there is nothing to load.

**Version control is deployment.** Your site history is your Git history. Rolling back a bad change is `git revert`. There is no separate deploy pipeline, no CI/CD configuration, no staging environment. Push to `main` and the site updates.

**Themes are just CSS and Liquid templates.** Switching themes is changing one line in `_config.yml`. The content stays the same. The presentation layer is completely decoupled from the content layer.

### The tradeoffs

Jekyll is deliberately limited:

- **No dynamic content.** No login pages, no form submissions, no database queries. If you need any of those, you need a different tool.
- **Build time.** Large sites (1000+ pages) can take minutes to build. GitHub Pages has a soft limit of about 10 minutes.
- **Plugin restrictions.** GitHub Pages only supports a whitelist of Jekyll plugins. Custom plugins require building locally and pushing the `_site/` folder directly.
- **Ruby.** If you do need to run Jekyll locally for testing, you need a Ruby environment. This is not always pleasant.

For a portfolio or technical blog, none of these limitations matter. You are writing Markdown and publishing it. That is exactly what Jekyll was built for.

### The mental model

Think of Jekyll as a compiler:

```text
Source code (Markdown + YAML)  →  Compiler (Jekyll)  →  Binary (HTML/CSS)
```

GitHub Pages is the operating system that runs your binary. You ship source code, GitHub compiles and runs it. The analogy holds surprisingly well: you can compile locally for testing (`jekyll serve`), but you do not have to.

### Getting started in three files

You need exactly three things to have a working GitHub Pages site with Jekyll:

**1. `_config.yml`** — pick a theme and set metadata.

**2. `README.md`** (or `index.md`) — your homepage content.

**3. A repository named `username.github.io`** — GitHub Pages activates automatically for repos with this naming pattern.

Push those three things and you have a live website. Everything else — blog posts, additional pages, custom CSS — is incremental from there.
