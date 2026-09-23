# Lovable Blog

A static blog for a single author. Astro + TypeScript, Markdown content collections,
Sveltia CMS at `/admin`, Pagefind search, Giscus comments. Designed to be hosted via
Bolt.new/bolt.host — but **this repo has not itself been imported into Bolt.new or
published**; see [Status](#status) below.

No server, no database, no tracking scripts, no cookie banner, no CSS framework. Three
runtime dependencies. The only client-side JavaScript is a theme toggle, a copy-link
button, the search page and the comment widget.

Sibling projects, same feature set, each with independent content and git history and a
deliberately different visual design: [`localsme-blog/`](../localsme-blog) (GitHub Pages),
[`localsme-gitlab-blog/`](../localsme-gitlab-blog) (GitLab Pages),
[`localsme-cloudflare-blog/`](../localsme-cloudflare-blog) (Cloudflare Pages),
[`localsme-netlify-blog/`](../localsme-netlify-blog) (Netlify),
[`localsme-vzero-blog/`](../localsme-vzero-blog) (Vercel via v0.app),
[`localsme-firebase-blog/`](../localsme-firebase-blog) (Firebase Hosting).

## Documentation

| Document | What it covers |
| --- | --- |
| [docs/setup.md](docs/setup.md) | One-time setup — GitHub repo (done), Bolt.new hosting (done), CMS token, Giscus, contact form, author details |
| [docs/writing.md](docs/writing.md) | Writing and publishing posts, frontmatter reference, drafts, images |
| [docs/architecture.md](docs/architecture.md) | How the site is built and why the awkward parts are that way |
| [docs/deployment.md](docs/deployment.md) | CI/CD — Bolt.new/bolt.host, including what's still unverified about it |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Common failures, with the fix |
| [SECURITY.md](SECURITY.md) | Threat model, token hygiene, why deletion is not erasure |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | Ground rules for comments |
| [CLAUDE.md](CLAUDE.md) | Conventions for anyone (or anything) editing this repository |

## Commands

| Command | What it does |
| --- | --- |
| `npm install` | Install dependencies |
| `npm run dev` | Dev server at http://localhost:4321/ — **drafts visible** |
| `npm run build` | Production build into `dist/`, then Pagefind indexes it (`postbuild`) |
| `npm run preview` | Serve `dist/` — the only faithful test of search, `/admin/` and base paths |
| `npm run check` | TypeScript + Astro diagnostics |

### Windows: Smart App Control

If a build fails with *"An Application Control policy has blocked this file"*, Windows
Smart App Control is refusing Astro's unsigned native compiler binary. Install the WASI
fallback:

```bash
npm install --no-save --force @astrojs/compiler-binding-wasm32-wasi
```

Re-run it after every `npm install` or `npm ci`. CI on Linux is unaffected. Details in
[troubleshooting.md](docs/troubleshooting.md).

## Writing a post

Open `/admin/` → **New Post** → uncheck **Draft** → **Save**. That commits to `main` on
GitHub, but does **not** by itself update the live bolt.host site — reopen the Bolt.new
project and click **Publish** to actually ship it. See
[docs/deployment.md](docs/deployment.md).

Or write the file directly — posts are Markdown in `src/content/blog/`, and the filename
is the URL slug:

```yaml
---
title: "Post title"
description: "One or two sentences, used for cards, meta description and social."
date: "2026-09-02T09:00:00.000Z"
author: "Your Name"
featured_image: "../../assets/images/uploads/something.jpg"
category: "Engineering"
tags: [astro, performance]
draft: false
---
```

Frontmatter is validated at build time — a typo fails the build instead of shipping
broken output, and a failed build leaves the previous version live.

`draft: true` posts render in `npm run dev` and are dropped entirely from production:
no page, no feed entry, no archive listing, no search hit. Full reference in
[writing.md](docs/writing.md).

## Base-path safety

`base` is currently `/` (a guess pending an actual host) and a root-absolute `/foo/`
link happens to work. That is a coincidence of the placeholder config, not a licence to
hardcode paths: every internal link still goes through [`src/lib/url.ts`](src/lib/url.ts)
— `withBase()` for paths you author, `absFromBuiltPath()` for paths Astro produced,
`absUrl()` for absolute URLs.

To verify after any change, build and confirm every absolute URL points at this site:

```bash
grep -rhoE 'https?://[^"< ]+' dist --include=*.html --include=*.xml   | grep -v 'localsme.bolt.host' | sort -u
```

## Status

This is a from-scratch LocalSME fork of a sibling project's codebase, with its own git
history — see [`CLAUDE.md`](CLAUDE.md).

- [x] GitHub repository created and pushed — public, `LocalSME/lovable-blog`
- [ ] Not yet imported into Bolt.new or published. `astro.config.mjs`,
      `public/admin/config.yml` and `robots.txt` currently point at the placeholder
      domain `localsme.bolt.host` (this project family's brand-word.host-tld
      convention) rather than a real Bolt.new project URL — update all three once one
      exists, see [docs/setup.md](docs/setup.md#1-boltnew-import-and-publish)
- [ ] CMS access token, Giscus IDs, contact form endpoint, author details in `src/consts.ts`

The site also builds and runs locally (`npm run dev` / `npm run build` / `npm run
preview`) independent of all of the above. Full checklist in
[docs/setup.md](docs/setup.md).
