# Deployment

> **Live** at <https://localsme.bolt.host>, imported and published via Bolt.new
> 2026-09-24. **Saving a post in the CMS does not update the live site by itself** —
> see below.

**GitHub repo:** `LocalSME/bolt-blog` — public, pushed, what the CMS commits to.
**Bolt.new project:** imported from `https://bolt.new/~/github.com/LocalSME/bolt-blog`,
published from Bolt's own editor to `localsme.bolt.host`.

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

### Confirmed: Bolt's Publish does NOT run the `postbuild` Pagefind step

Checked 2026-09-24 after enabling `FEATURES.search`: `/pagefind/pagefind-ui.js` 404s on
the live `localsme.bolt.host` site even though a local `npm run build` produces it fine.
The same 404 was independently confirmed on `creativedigitalgrowth.bolt.host` (the
sibling this repo was forked from, also Bolt-hosted) — this is a platform-wide Bolt.new
limitation, not something specific to this repo, and not something a script change can
fix from inside the repo (most likely: WebContainer's browser-sandboxed Node can't run
Pagefind's compiled Rust binary, so the `postbuild` script silently never completes,
regardless of whether the runner even reaches it).

**Workaround in place: the Pagefind index is committed as a static asset.** Unlike
`dist/`, `public/` is not gitignored and is what Astro copies verbatim into `dist/`
during the (working) `astro build` step — no native binary execution required for that
part. So `public/pagefind/` in this repo holds a pre-built snapshot of the index,
regenerated and committed by hand:

```
npm run build                        # regenerates dist/pagefind from current content
rm -rf public/pagefind
cp -r dist/pagefind public/pagefind
git add public/pagefind && git commit -m "Refresh Pagefind index" && git push
```

**This must be re-run and re-pushed before every Bolt Publish that adds/edits/removes a
post**, in addition to the existing CMS-save → reopen-Bolt → Publish flow above —
otherwise search results go stale (or, before the index existed at all, 404 outright).
If Bolt's own build ever does manage to run `postbuild` successfully in the future, it
would just overwrite this snapshot with an equally fresh one — harmless either way.

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
