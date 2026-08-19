# Hugo + Blowfish Blog Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the hand-written `index.html` at `powens.github.io` with a Hugo blog using the Blowfish theme, deployed to GitHub Pages by GitHub Actions on every push to `master`.

**Architecture:** A stock Hugo site. Blowfish is a git submodule at `themes/blowfish`, pinned to tag `v3.1.0`, and is never edited. All customisation lives in `config/_default/*.toml`. Content is Markdown under `content/`. CI installs a pinned Hugo binary, builds, and uploads the result as a GitHub Pages artifact.

**Tech Stack:** Hugo v0.165.0, Blowfish v3.1.0, GitHub Actions, GitHub Pages.

**Spec:** `docs/superpowers/specs/2026-08-19-hugo-blowfish-blog-design.md`

## Note on testing

This project has no unit test suite, and adding one would be inappropriate — there is no application code, only content and configuration. The TDD cycle is therefore adapted rather than skipped: **every task states an observable failing condition, then makes it pass.** The build itself is the test harness. `hugo build` writes to `public/`, and each task asserts on files and strings in that output. Do not skip the "verify it fails first" steps; they are what prove the assertion is actually testing something.

## Global Constraints

- Hugo version is exactly **0.165.0** locally and in CI. Blowfish v3.1.0 requires Hugo >= 0.162.0.
- Blowfish is pinned to tag **v3.1.0**. Never edit any file under `themes/blowfish/`; that directory is a submodule.
- The repository default branch is **`master`**, not `main`. Upstream samples say `main` and must be adapted.
- Site title is exactly `Patrick Owens`.
- Tagline is exactly `Software, radio, and small plastic soldiers.` (including the trailing period).
- GitHub link target is exactly `https://github.com/powens`.
- Never create a `static/` directory in this plan — git cannot track an empty directory and there is no asset for it yet.
- Never touch DNS, custom domains, or `CNAME`. The site lives at `https://powens.github.io`.
- `public/`, `resources/_gen/`, and `.hugo_build.lock` are build output and must stay untracked.

---

### Task 1: Clear the old site and scaffold the repo skeleton

**Files:**
- Delete: `index.html`, `img/` (11 files), `font/Tau.otf`
- Create: `.gitignore`
- Modify: `README.md`
- Keep untouched: `LICENSE`, `docs/`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: a clean working tree with `.gitignore` in place, ready for a Hugo site. Later tasks assume `LICENSE` and `docs/` still exist and that no `index.html` remains at the repo root.

- [ ] **Step 1: Install Hugo and verify the version satisfies the Blowfish floor**

```bash
brew install hugo
hugo version
```

Expected: version string reporting `v0.165.0` or newer. If Homebrew installs something older than `v0.162.0`, stop — Blowfish v3.1.0 will not build. Run `brew upgrade hugo` and re-check.

- [ ] **Step 2: Confirm the old site is recoverable before deleting it**

```bash
git log --oneline -1 a4fa12b
git show --stat a4fa12b:index.html | head -5
```

Expected: commit `a4fa12b` resolves and `index.html` is readable at that commit. This is the safety net the spec relies on. If this fails, stop and investigate rather than deleting.

- [ ] **Step 3: Delete the old site files**

```bash
git rm -r --quiet index.html img font
git status --short
```

Expected: deletions staged for `index.html`, all 11 files under `img/`, and `font/Tau.otf`. `LICENSE`, `README.md`, and `docs/` untouched.

- [ ] **Step 4: Create `.gitignore`**

```gitignore
# Hugo build output
public/
resources/_gen/
.hugo_build.lock

# Hugo module / theme caches
.hugo/

# macOS
.DS_Store
```

- [ ] **Step 5: Rewrite `README.md`**

````markdown
# powens.github.io

Personal site and blog for Patrick Owens, built with [Hugo](https://gohugo.io)
and the [Blowfish](https://blowfish.page) theme, deployed to GitHub Pages.

Live at <https://powens.github.io>.

## Local development

Clone with submodules, since the theme is one:

```bash
git clone --recurse-submodules git@github.com:powens/powens.github.io.git
```

If you already cloned without them:

```bash
git submodule update --init --recursive
```

Then run the dev server:

```bash
hugo server -D
```

`-D` includes drafts. The site is served at <http://localhost:1313>.

## Writing a post

```bash
hugo new content posts/my-post-title/index.md
```

Edit the front matter, set `draft = false` when ready, and push to `master`.
GitHub Actions builds and deploys automatically.

## Requirements

- Hugo v0.165.0 or newer (Blowfish v3.1.0 requires >= 0.162.0)
````

- [ ] **Step 6: Verify the working tree is as expected**

```bash
ls -A
```

Expected exactly: `.git`, `.gitignore`, `LICENSE`, `README.md`, `docs`. No `index.html`, no `img`, no `font`.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "Remove hand-written Tau site, scaffold for Hugo

The old index.html and its assets remain recoverable at a4fa12b."
```

---

### Task 2: Add Blowfish as a pinned submodule

**Files:**
- Create: `.gitmodules`
- Create: `themes/blowfish/` (submodule, not tracked file-by-file)

**Interfaces:**
- Consumes: the clean tree from Task 1.
- Produces: `themes/blowfish/` present on disk, pinned to tag `v3.1.0`. Task 3 sets `theme = "blowfish"` and depends on that exact directory name (lowercase).

- [ ] **Step 1: Verify the theme is absent (the failing condition)**

```bash
ls themes/blowfish 2>&1
```

Expected: `No such file or directory`. This is the condition Task 3's config would fail on.

- [ ] **Step 2: Add the submodule**

```bash
git submodule add --depth=1 https://github.com/nunocoracao/blowfish.git themes/blowfish
```

- [ ] **Step 3: Pin it to tag v3.1.0**

`--depth=1` clones only the default branch tip, so the tag must be fetched explicitly before checkout.

```bash
git -C themes/blowfish fetch --depth=1 origin tag v3.1.0
git -C themes/blowfish checkout v3.1.0
```

- [ ] **Step 4: Verify the pin and that the theme's assets are present**

```bash
git -C themes/blowfish describe --tags
ls themes/blowfish/assets/css/compiled/ | head
ls themes/blowfish/assets/icons/github.svg themes/blowfish/assets/icons/rss.svg
```

Expected: `v3.1.0`; a non-empty `compiled/` directory (this is the precompiled Tailwind CSS that makes Dart Sass unnecessary); and both icon files present, which Task 3's profile links depend on.

- [ ] **Step 5: Confirm no SCSS exists, justifying the CI simplification**

```bash
find themes/blowfish -name '*.scss' | wc -l
```

Expected: `0`. This is the evidence behind dropping the Dart Sass step in Task 5. If this is ever non-zero after a theme bump, that step must be restored.

- [ ] **Step 6: Commit**

```bash
git add .gitmodules themes/blowfish
git commit -m "Add Blowfish theme as submodule pinned to v3.1.0"
```

---

### Task 3: Configure the site

**Files:**
- Create: `config/_default/hugo.toml`
- Create: `config/_default/languages.en.toml`
- Create: `config/_default/markup.toml`
- Create: `config/_default/menus.en.toml`
- Create: `config/_default/params.toml`

**Interfaces:**
- Consumes: `themes/blowfish/` at v3.1.0 from Task 2.
- Produces: a buildable site. Task 4 relies on the `tags` taxonomy being declared here, on `posts` being the content section name, and on `mainSections = ["posts"]` for the homepage recent-posts list.

- [ ] **Step 1: Confirm the site does not build yet (the failing condition)**

```bash
hugo build 2>&1 | tail -5
```

Expected: FAIL — Hugo reports no configuration file / unable to locate config. Nothing is written to `public/`.

- [ ] **Step 2: Create `config/_default/hugo.toml`**

`[caches.images]` is required by the CI workflow in Task 5, which passes `--cacheDir`; without it the image cache lands outside the cached directory and the cache step does nothing.

```toml
theme = "blowfish"
baseURL = "https://powens.github.io/"
defaultContentLanguage = "en"

enableRobotsTXT = true
enableEmoji = true
summaryLength = 0

buildDrafts = false
buildFuture = false

[pagination]
  pagerSize = 10

[taxonomies]
  tag = "tags"

[sitemap]
  changefreq = "weekly"
  filename = "sitemap.xml"
  priority = 0.5

[outputs]
  home = ["HTML", "RSS", "JSON"]

[imaging]
  anchor = "Center"

[caches]
  [caches.images]
    dir = ":cacheDir/images"
```

Note on `languageCode`: this key is deliberately absent from `hugo.toml`. Hugo 0.158 deprecated it and it was inert for this site — `locale = "en"` in `languages.en.toml` is what actually drives `<html lang>` and the RSS feed's language. Do not re-add `languageCode`; it has no effect and only invites drift between two sources of truth.

Note on `[taxonomies]`: Blowfish's own default declares four taxonomies (`tag`, `category`, `author`, `series`). This site declares only `tags`, per the spec's scope. If the build in Step 7 fails with an error naming `categories`, `authors`, or `series`, restore Blowfish's full block rather than debugging templates:

```toml
[taxonomies]
  tag = "tags"
  category = "categories"
  author = "authors"
  series = "series"
```

`[outputs] home` must keep `JSON` — Blowfish's search index is a JSON output of the home page, and `enableSearch` in `params.toml` silently produces a broken search box without it.

- [ ] **Step 3: Create `config/_default/languages.en.toml`**

```toml
disabled = false
locale = "en"
label = "English"
weight = 1
title = "Patrick Owens"

[params]
  displayName = "EN"
  isoCode = "en"
  rtl = false
  dateFormat = "2 January 2006"
  description = "Software, radio, and small plastic soldiers."
  copyright = "© Patrick Owens"

[params.author]
  name = "Patrick Owens"
  headline = "Software, radio, and small plastic soldiers."
  links = [
    { github = "https://github.com/powens" },
    { rss = "/index.xml" },
  ]
```

Each entry in `links` is a single-key table whose key names an SVG in `themes/blowfish/assets/icons/`. Both `github` and `rss` were confirmed present in Task 2 Step 4. An unrecognised key renders no icon and fails silently, so do not invent keys.

- [ ] **Step 4: Create `config/_default/markup.toml`**

Copied from Blowfish's default. The theme documents these as required for correct rendering.

```toml
[goldmark]
  [goldmark.parser]
    wrapStandAloneImageWithinParagraph = false

    [goldmark.parser.attribute]
      block = true

  [goldmark.renderer]
    unsafe = true

[highlight]
  noClasses = false

[tableOfContents]
  startLevel = 2
  endLevel = 4
```

- [ ] **Step 5: Create `config/_default/menus.en.toml`**

```toml
# -- Main menu (header) --
[[main]]
  name = "Posts"
  pageRef = "posts"
  weight = 10

[[main]]
  name = "Tags"
  pageRef = "tags"
  weight = 20

# -- Footer menu --
[[footer]]
  name = "RSS"
  url = "/index.xml"
  weight = 10
```

- [ ] **Step 6: Create `config/_default/params.toml`**

`showTaxonomies = true` is deliberate — Blowfish defaults it to `false`, which would hide tags on posts and make Task 4's tag verification misleading.

```toml
colorScheme = "blowfish"
defaultAppearance = "dark"
autoSwitchAppearance = true

enableSearch = true
enableCodeCopy = true

mainSections = ["posts"]

disableImageOptimization = false
disableTextInHeader = false

fingerprintAlgorithm = "sha512"

[header]
  layout = "basic"

[footer]
  showMenu = true
  showCopyright = true
  showThemeAttribution = true
  showAppearanceSwitcher = true
  showScrollToTop = true

[homepage]
  layout = "profile"
  showRecent = true
  showRecentItems = 5
  showMoreLink = true
  showMoreLinkDest = "/posts/"

[article]
  showDate = true
  showAuthor = false
  showBreadcrumbs = false
  showDraftLabel = true
  showHeadingAnchors = true
  showPagination = true
  showReadingTime = true
  showTableOfContents = true
  showTaxonomies = true
  showTags = true
  showWordCount = true

[list]
  showSummary = true
  groupByYear = true

[taxonomy]
  showTermCount = true

[term]
  showTableOfContents = false
```

- [ ] **Step 7: Verify the site now builds**

```bash
hugo build
```

Expected: PASS — exit code 0, no `ERROR` lines. Hugo prints a page count summary.

- [ ] **Step 8: Verify the title and tagline actually reach the rendered page**

```bash
grep -c "Patrick Owens" public/index.html
grep -c "Software, radio, and small plastic soldiers." public/index.html
```

Expected: both counts >= 1. A zero here means the config is being parsed but not applied — most likely a misplaced file under `config/_default/`.

- [ ] **Step 9: Commit**

```bash
git add config
git commit -m "Configure Hugo site with Blowfish theme

Title, tagline, tags-only taxonomy, profile homepage, GitHub and RSS links."
```

---

### Task 4: Add content and prove tags and the feed work

**Files:**
- Create: `archetypes/default.md`
- Create: `content/_index.md`
- Create: `content/posts/_index.md`
- Create: `content/posts/hello-world/index.md`

**Interfaces:**
- Consumes: the `tags` taxonomy and `mainSections = ["posts"]` from Task 3.
- Produces: `public/posts/hello-world/index.html`, `public/tags/meta/index.html`, and `public/index.xml`. Task 6's live-site check asserts on these same paths.

- [ ] **Step 1: Verify the tag term page and posts section do not exist yet (the failing condition)**

```bash
rm -rf public
hugo build >/dev/null
ls public/posts 2>&1
ls public/tags/meta 2>&1
```

Expected: both report `No such file or directory`. Note that `public/tags` itself already exists at this point — Hugo generates an empty taxonomy list page purely because `[taxonomies] tag = "tags"` is declared in `config/_default/hugo.toml`, regardless of whether any content uses it. That list page is not a meaningful fail-first signal, so this check targets the *term* page (`public/tags/meta`) instead, alongside the `posts` section, which genuinely does not exist until content is added — which is exactly why the assertions in Step 7 are meaningful.

- [ ] **Step 2: Create `archetypes/default.md`**

This is the template `hugo new content` uses, so every future post starts with the right front matter.

```markdown
+++
title = "{{ replace .File.ContentBaseName "-" " " | title }}"
date = {{ .Date }}
draft = true
tags = []
+++
```

- [ ] **Step 3: Create `content/_index.md`**

The homepage. Blowfish's `profile` layout renders the author block from config; this file supplies any prose beneath it.

```markdown
+++
title = "Patrick Owens"
+++
```

- [ ] **Step 4: Create `content/posts/_index.md`**

```markdown
+++
title = "Posts"
description = "Writing about software, radio, and the occasional dice roll."
+++
```

- [ ] **Step 5: Create `content/posts/hello-world/index.md`**

`draft = false` is required — the CI build does not pass `-D`, so a draft would build locally under `hugo server -D` but vanish in production. The `tags` list is what makes the Step 7 taxonomy assertion real.

```markdown
+++
title = "Hello World"
date = 2026-08-19T00:00:00-07:00
draft = false
tags = ["meta"]
+++

This site now runs on [Hugo](https://gohugo.io) with the
[Blowfish](https://blowfish.page) theme, replacing the hand-written page that
lived here before.

The old page is not gone, only unpublished — it survives in this repository's
git history.

More soon.
```

- [ ] **Step 6: Rebuild**

```bash
rm -rf public
hugo build
```

Expected: PASS — exit 0, and the page count is higher than in Task 3 Step 7.

- [ ] **Step 7: Verify the post, its tag page, and the feed all exist**

```bash
test -f public/posts/hello-world/index.html && echo "post OK"
test -f public/tags/meta/index.html        && echo "tag term OK"
test -f public/tags/index.html             && echo "tag list OK"
test -f public/index.xml                   && echo "feed OK"
test -f public/index.json                  && echo "search index OK"
```

Expected: all five lines print. A missing `tag term` means `tags` is not declared in `hugo.toml`; a missing `feed` means `RSS` was dropped from `[outputs] home`.

- [ ] **Step 8: Verify the feed and the post page have real content**

```bash
grep -c "Hello World" public/index.xml
grep -c "meta" public/posts/hello-world/index.html
```

Expected: both >= 1. The first proves the post is syndicated rather than the feed merely existing; the second proves `showTaxonomies = true` took effect and tags render on the post.

- [ ] **Step 9: Confirm build output stayed untracked**

```bash
git status --porcelain --ignored public | head -3
git status --short
```

Expected: `public/` shows as ignored, and `git status --short` lists only the four new content files. If `public/` appears as untracked-but-not-ignored, `.gitignore` from Task 1 is wrong.

- [ ] **Step 10: Commit**

```bash
git add archetypes content
git commit -m "Add homepage, posts section, and first post"
```

---

### Task 5: Add the GitHub Actions deploy workflow

**Files:**
- Create: `.github/workflows/hugo.yaml`

**Interfaces:**
- Consumes: the buildable site from Tasks 3 and 4, and `.gitmodules` from Task 2.
- Produces: a workflow that runs on push to `master`. Task 6 depends on the job names `build` and `deploy` and on the `github-pages` environment.

This is Hugo's official workflow with four changes, all justified in the spec: trigger on `master`; time zone set to `America/Vancouver`; the Dart Sass install step removed (proved unnecessary in Task 2 Step 5); and the Go and Node steps removed. Upstream already guards those two with `hashFiles('go.mod')` and `hashFiles('package-lock.json')`, which match only at the repository root — the theme submodule's own `go.mod` and `package.json` never match — so removing them changes nothing at runtime and only removes noise.

- [ ] **Step 1: Verify no workflow exists yet (the failing condition)**

```bash
ls .github/workflows 2>&1
```

Expected: `No such file or directory`.

- [ ] **Step 2: Create `.github/workflows/hugo.yaml`**

```yaml
name: Build and deploy
on:
  push:
    branches:
      - master
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.165.0
      TZ: America/Vancouver
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0
          lfs: false

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Create a local tools directory
        run: |
          mkdir -p "${HOME}/.local"

      - name: Install Hugo
        run: |
          echo "Installing Hugo ${HUGO_VERSION}..."
          curl -sfL --output-dir "${{ runner.temp }}" -O "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "${{ runner.temp }}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"

      - name: Log tool versions
        run: |
          hugo version

      - name: Configure Git
        run: |
          git config --global core.quotepath false

      - name: Fetch full Git history
        run: |
          if [[ $(git rev-parse --is-shallow-repository) == true ]]; then
            echo "Fetching full Git history..."
            git fetch --unshallow
          fi

      - name: Initialize Git submodules
        run: |
          if [[ -f .gitmodules ]]; then
            echo "Initializing Git submodules..."
            git submodule update --init --recursive
          fi

      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v6
        with:
          path: ${{ runner.temp }}/.cache/hugo
          key: hugo-${{ github.run_id }}
          restore-keys: hugo-

      - name: Build
        run: |
          echo "Building the project..."
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/.cache/hugo"

      - name: Cache save
        uses: actions/cache/save@v6
        with:
          path: ${{ runner.temp }}/.cache/hugo
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          include-hidden-files: false
          path: ./public
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

- [ ] **Step 3: Verify the YAML parses**

```bash
python3 -c "import yaml,sys; d=yaml.safe_load(open('.github/workflows/hugo.yaml')); print('jobs:', list(d['jobs'])); print('branches:', d[True]['push']['branches'])"
```

Expected: `jobs: ['build', 'deploy']` and `branches: ['master']`. (PyYAML parses the bare `on:` key as boolean `True`; that is a quirk of the parser, not a problem with the file.)

- [ ] **Step 4: Verify the four intended deviations from upstream**

```bash
grep -c 'dart-sass\|DART_SASS'   .github/workflows/hugo.yaml
grep -c 'setup-go\|setup-node'   .github/workflows/hugo.yaml
grep -c 'submodules: recursive'  .github/workflows/hugo.yaml
grep -c 'America/Vancouver'      .github/workflows/hugo.yaml
```

Expected: `0`, `0`, `1`, `1` in that order.

- [ ] **Step 5: Reproduce the CI build command locally**

This runs exactly what CI will run, so a failure surfaces here rather than after a push.

```bash
rm -rf public
hugo build --gc --minify --baseURL "https://powens.github.io/"
test -f public/index.xml && test -f public/tags/meta/index.html && echo "CI-equivalent build OK"
```

Expected: exit 0 and `CI-equivalent build OK`. If this fails but Task 4's plain `hugo build` passed, the cause is `--minify` or `--gc`; fix it before pushing rather than debugging in CI.

- [ ] **Step 6: Commit**

```bash
git add .github
git commit -m "Add GitHub Actions workflow to build and deploy to Pages

Hugo's official workflow, adapted: triggers on master, Vancouver time
zone, and without the Dart Sass, Go, and Node steps that this theme
does not need."
```

---

### Task 6: Deploy and verify the live site

**Files:** none created or modified. This task is push, one owner-performed setting, and verification.

**Interfaces:**
- Consumes: everything from Tasks 1-5.
- Produces: a live site. Terminal task.

- [ ] **Step 1: Verify the full history is coherent before pushing**

```bash
git log --oneline master...origin/master
git status --short
```

Expected: five new commits (Tasks 1-5) listed, and a clean working tree.

- [ ] **Step 2: Confirm the Pages source setting with the repository owner**

**STOP. This step is performed by the repository owner, not by an agent.**

Ask the owner to set repo Settings -> Pages -> Source to **GitHub Actions**. Until this is done, the `deploy` job fails with a permissions or "Pages not enabled" error even though `build` succeeds.

Current state can be checked with:

```bash
gh api repos/powens/powens.github.io/pages --jq '.build_type' 2>&1
```

Expected once set: `workflow`. A `404` means Pages is not enabled yet; anything else (for example `legacy`) means the source is still branch-based and must be changed.

- [ ] **Step 3: Push**

```bash
git push origin master
```

- [ ] **Step 4: Watch the run to completion**

```bash
gh run watch --exit-status $(gh run list --branch master --limit 1 --json databaseId --jq '.[0].databaseId')
```

Expected: both jobs succeed, exit status 0. If `build` fails on a missing theme, the checkout is not fetching submodules — re-check Task 5 Step 2. If it fails on a missing `sass` binary, restore the Dart Sass step from the spec.

- [ ] **Step 5: Verify the live site serves each required artifact**

```bash
for p in "" "posts/hello-world/" "tags/" "tags/meta/" "index.xml"; do
  printf '%-24s %s\n' "/$p" "$(curl -s -o /dev/null -w '%{http_code}' "https://powens.github.io/$p")"
done
```

Expected: `200` on all five lines. A `404` on the root with `200` elsewhere usually means Pages has not finished propagating; wait a minute and retry before investigating.

- [ ] **Step 6: Verify the deployed HTML has the right content and links**

```bash
curl -s https://powens.github.io/ | grep -c "Software, radio, and small plastic soldiers."
curl -s https://powens.github.io/ | grep -c "https://github.com/powens"
curl -s https://powens.github.io/index.xml | grep -c "Hello World"
```

Expected: all three >= 1. These confirm the tagline, the GitHub link, and the feed respectively — the three things the spec's verification section calls for beyond mere page existence.

- [ ] **Step 7: Confirm the old site is gone but recoverable**

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://powens.github.io/img/astley.gif
git show a4fa12b:index.html | head -3
```

Expected: `404` from the live URL, and readable HTML from git. This is the spec's clean-slate decision holding in both directions.

- [ ] **Step 8: Report completion**

Report to the owner: the live URL, the run URL, and a reminder that their GitHub profile's website field still points at the expired `padraig.io` and is now a dead link.

---

## Rollback

If the deploy is broken and needs to be reverted quickly, the old site can be restored to a working tree with:

```bash
git checkout a4fa12b -- index.html img font
```

Note that this only restores the files; with Pages set to build from Actions, serving them again also requires either reverting the workflow or switching the Pages source back to a branch.
