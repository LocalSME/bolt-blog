# Deployment

> **Not yet live.** This LocalSME fork has not itself been imported into Bolt.new or
> published — there is no real Bolt.new project or bolt.host URL for it yet. This page
> describes the Bolt.new/bolt.host mechanism this project is designed for (and the
> behaviour confirmed on the project it was forked from), as reference for whoever sets
> that up. It replaced the Cloudflare Pages mechanism this page described before
> (inherited unmodified from the template this repo was copied from). **Once it's live,
> remember: saving a post in the CMS does not update the live site by itself** — see
> below.

**GitHub repo:** `LocalSME/bolt-blog` — public, pushed, what the CMS commits to.
**Bolt.new project:** not yet created. Once one exists, it would be imported from this
repo via a `https://bolt.new/~/github.com/LocalSME/bolt-blog` URL, then published
from Bolt's own editor to whatever `<project>.bolt.host` URL Bolt assigns —
`astro.config.mjs`, `public/admin/config.yml` and `public/robots.txt` would all need
updating off the current `localsme.bolt.host` placeholder to match.

## How it works

```
Expect this (confirmed on the project this was forked from):
  bolt.new/~/github.com/LocalSME/bolt-blog
    └─ Bolt.new imports the repo into a WebContainer-based editor session
       └─ user clicks "Publish" inside Bolt's editor
          └─ Bolt builds and deploys the current in-editor workspace to
             the assigned <project>.bolt.host URL

  Bolt.new → GitHub push-back is real, not just a one-time import snapshot: on the
  project this was forked from, a commit ("Updated package-lock.json", authored via
  GitHub's own committer identity for the account that owned that project) landed on
  `main` within minutes of import, reflecting an `npm install` Bolt ran inside its own
  WebContainer. Bolt maintains a live, writable connection back to the GitHub repo.

Expect this too — the reverse direction does NOT auto-deploy:
  git push to main from outside Bolt (e.g. a CMS save)
    └─ does NOT republish bolt.host by itself, unlike Cloudflare/Netlify's Git
       integrations. Reproduced on the project this was forked from: a CMS save landed
       as a commit on GitHub within seconds, but the live site kept showing only the
       previous content with no new build triggered.
    └─ the fix is to reopen the project in Bolt.new and click "Publish" again —
       that pulls the latest `main` into the editor session and republishes it
```

**So publishing a CMS post will be a two-step process here, unlike every other
sibling**: save in the CMS (commits to `main`), then separately open Bolt.new and click
Publish. There is no GitHub Actions workflow and no repository secrets involved in the
Bolt path either way.

### Why the npm `postbuild` hook matters

Whatever builds this project needs to run `npm run build` rather than `astro build`
directly, so the `postbuild` script fires: npm runs `pagefind --site dist` right after
Astro finishes, and the search index ends up inside `dist/` before it gets published. No
extra build step to configure, and no way to deploy a site whose search index is stale.
Whether Bolt's Publish action actually runs the full `npm run build` (postbuild
included) rather than just `astro build` has not been checked — if search ever comes up
empty on the live site, start there.

## What's configurable, and where

There is no `wrangler.toml`, no GitHub Actions workflow file, and no repository secrets
in this repo for deployment — Bolt.new's build/publish settings, if any are exposed at
all, would live inside the Bolt.new project's own UI, not in a file here. Not yet
explored; check Bolt's project settings directly if a build setting ever needs changing.

| Setting | Value here |
| --- | --- |
| Node version | unconfirmed — matches whatever Bolt's WebContainer runtime uses |
| Environment variables | none required by this project today |

## Verifying a deployment

Bolt.new's own UI (the project's Publish/deploy history, if it has one) is unexplored —
check there first if a publish ever seems to fail silently.

A smoke test against the live site is the check that actually matters — it tests what
visitors get rather than what the local build produced:

```bash
B=https://localsme.bolt.host   # swap in the real <project>.bolt.host URL once one exists
for p in "" "blog/" "about/" "contact/" "search/" "admin/" "rss.xml" "sitemap-index.xml" "pagefind/pagefind-ui.js"; do
  echo "$(curl -s -o /dev/null -w '%{http_code}' -L "$B/$p")  /$p"
done
```

All should return `200`. Then confirm nothing leaked:

```bash
# drafts must be absent — swap in the slug of an actual draft post once one exists
curl -s -o /dev/null -w '%{http_code}\n' -L "$B/blog/<draft-slug>/"   # expect 404

# no root-absolute internal references
curl -s -L "$B/" | grep -ohE 'https?://[^"]+' | grep -v 'localsme.bolt.host' | sort -u
```

## Rollback

Not yet exercised. If Bolt.new keeps a publish history the way its Deployments-style UIs
often do, an older publish may be re-selectable from inside the project — unverified.
The from-source fallback that always works regardless of what Bolt exposes: revert the
bad commit in GitHub, then re-import/re-open the Bolt project and Publish again.

```bash
git revert <sha>
git push
```

Unlike the Cloudflare/Netlify siblings, pushing this alone is **confirmed not** to
redeploy — see above. "Revert and push" is step one of two; "reopen Bolt and Publish" is
the required step two.

## Local equivalents

```bash
npm run build     # what CI runs, including Pagefind
npm run preview   # serves dist/ — the only faithful local test of search and base paths
```

`npm run dev` does **not** exercise search (no index), the `/admin/` directory index, or
draft exclusion. Use `preview` before assuming a deploy will behave.

## Access

Two independent access paths, not one.

**GitHub.** Pushing requires write access to `LocalSME/bolt-blog`.
Changing repository settings — Discussions, collaborators, and which GitHub Apps are
installed — requires **admin**, held by `LocalSME`. The `LocalSME`
account has Write only — same pattern as the sibling GitHub Pages repo.

```bash
gh api repos/LocalSME/bolt-blog --jq '.permissions'
```

**Bolt.new.** Separately, once a project exists, whoever is signed into the Bolt.new
account that imported and published it controls what's actually live — republishing,
and any build/env settings Bolt exposes. GitHub write access alone cannot make bolt.host
redeploy if pushes turn out not to sync automatically (see above); Bolt account access
alone cannot change what code exists in the GitHub repo unless that account also pushes
back to it. Both matter, independently — and which Bolt.new account will hold this
project hasn't been decided yet.
