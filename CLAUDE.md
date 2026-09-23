# Working in this repository

A solo-author static blog: Astro 7 + TypeScript. Structurally copied from the sibling
Cloudflare Pages blog (`LocalSME/cloudflare-blog`) as this family's seventh member,
alongside the GitHub Pages, GitLab Pages, Netlify, Vercel (`vzero-blog`) and Firebase
Hosting siblings — not a mirror of any of them, no shared content, no shared git
history. This is an independent LocalSME fork with its own fresh git history, cloned
into its own directory rather than reusing the original working copy. Full detail in
[`docs/architecture.md`](docs/architecture.md).

**Hosting is meant to be Bolt.new/bolt.host**, the same as the project this was forked
from, but **this fork has not itself been imported into Bolt.new or published** —
`astro.config.mjs`'s `site` (and the matching values in `public/admin/config.yml` and
`public/robots.txt`) currently point at the placeholder domain `localsme.bolt.host`
rather than a real project URL. The working name "lovable-blog" reflects the platform
originally asked about (lovable.dev), a prompt-first React/Vite app builder — that
platform's own GitHub sync was never tried, since Bolt.new turned out to be the tool
actually used upstream.

Expect this Bolt.new behaviour once/if this fork is imported and published (confirmed on
the project it was forked from): Bolt.new → GitHub push-back is real — it commits a
`package-lock.json` update from its own `npm install` shortly after import — but the
reverse does **not** hold. A `git push` made outside Bolt (e.g. a CMS save) does **not**
get pulled in and redeployed automatically; it needs a manual reopen-and-Publish inside
Bolt's editor. See [`docs/deployment.md`](docs/deployment.md).

## Development

Start the dev server in background mode:

```
astro dev --background
```

Manage it with `astro dev stop`, `astro dev status`, `astro dev logs`.

```bash
npm run dev      # localhost:4321/ — drafts visible
npm run build    # production build + Pagefind index
npm run preview  # serves dist/ — the only faithful test of search and base paths
npm run check    # TypeScript + Astro diagnostics; keep this at 0 errors
```

**On this Windows machine**, Smart App Control blocks Astro's native compiler binary.
After every `npm install` or `npm ci`:

```bash
npm install --no-save --force @astrojs/compiler-binding-wasm32-wasi
```

## Rules that are easy to get wrong

**Never write a root-absolute internal path.** It happens to work today — this is a
root-served site with `base: '/'` — but that is hosting, not design. Use the helpers in
`src/lib/url.ts` so the site can move again without a rewrite:

| Helper | For |
| --- | --- |
| `withBase(p)` | paths you author — `/about/` → `/about/` here, `/blog/about/` under a base |
| `absFromBuiltPath(p, site)` | paths Astro produced (`Astro.url.pathname`, `ImageMetadata.src`, `paginate()` URLs) — **already based** |
| `absUrl(p, site)` | absolute URL from a path you author |

Do not try to collapse these into one function that detects whether a path "already has
the base". That was tried on a sibling project and shipped two bugs: `/blog/my-post/`
is genuinely ambiguous because a `/blog/` route sits under a `/blog/` base.

**`paginate()` URLs already include the base.** Passing them through `withBase()`
doubles it the moment a base is configured. `Pagination.astro` takes them raw.

**Query posts through `getPosts()`** in `src/lib/posts.ts`, never `getCollection`
directly. That single call is where drafts are filtered out of production builds and
where date ordering happens.

**Frontmatter image paths are relative to the Markdown file** —
`../../assets/images/uploads/…` — because `image()` resolves them that way and the CMS
is configured to write exactly that. Both `media_folder` and `public_folder` in
`public/admin/config.yml` must stay in sync with wherever posts live.

**Site-wide settings live in `src/consts.ts` and nowhere else.** If you find yourself
hardcoding a title, an author name or a page size, put it there instead.

**The CMS schema and the Zod schema must match.** `public/admin/config.yml` field names
and `src/content.config.ts` are one contract; changing either alone breaks editing or
breaks the build.

**`public/admin/config.yml` is YAML.** Quote any string containing `: ` — an unquoted
colon-space silently breaks the whole CMS.

**The content repo (GitHub) and the host are meant to stay two separate systems**,
bridged by whatever sync the host offers — that's the pattern every sibling follows
(Cloudflare/Netlify's own dashboard integration, GitHub Actions for GitHub
Pages/Firebase, GitLab CI, a v0.app→Vercel connection). Sveltia CMS commits to the
GitHub repo named in `public/admin/config.yml` regardless. **For this sibling
specifically, a CMS save does NOT reach the live site by itself** — confirmed
2026-09-09: Bolt.new pulls from GitHub only when its own editor is reopened and
Publish is clicked again, unlike every other sibling's automatic Git-integration
deploy. Republish manually in Bolt after every CMS save.

## Before calling a change done

```bash
npm run check    # expect 0 errors
npm run build
grep -rhoE 'https?://[^"< ]+' dist --include=*.html | grep -v 'localsme.bolt.host' | sort -u
```

The grep must print only genuinely external URLs (giscus, google maps, unpkg). If the
change is visible in a browser, verify with `npm run preview` rather than `npm run dev`
— search, `/admin/` and draft exclusion all behave differently between the two.

## Deployment

**Not yet live.** This repo has not been imported into Bolt.new or published. Once it
is: pushing to `main` — including a CMS save — will **not** trigger a rebuild by itself;
someone has to reopen the Bolt.new project and click Publish again, the same manual
republish behaviour confirmed on the project this was forked from. See
[`docs/deployment.md`](docs/deployment.md).

Local git authenticates as `LocalSME`, the same account used for every LocalSME
sibling — that account's access would be unrelated to whoever controls the Bolt.new
project itself, once one exists.

## Documentation

Full docs: https://docs.astro.build

- [Routing and dynamic routes](https://docs.astro.build/en/guides/routing/)
- [Content collections](https://docs.astro.build/en/guides/content-collections/)
- [Images](https://docs.astro.build/en/guides/images/)
- [Astro components](https://docs.astro.build/en/basics/astro-components/)

Bolt.new-specific: https://support.bolt.new/
