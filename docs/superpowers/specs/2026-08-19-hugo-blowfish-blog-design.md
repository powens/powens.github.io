# Replace powens.github.io with a Hugo + Blowfish blog

**Date:** 2026-08-19
**Status:** Approved, ready for implementation planning

## Problem

`powens.github.io` is a single hand-written `index.html`: a Warhammer 40k
Tau-themed joke page with animated gifs, an autoplaying midi, a hit counter, and
a pasted army list. There is no way to publish writing to it.

Replace it with a static site generator blog that supports dated posts, tags,
and a feed, deployed automatically on push.

## Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Generator | Hugo v0.165.0 | Chosen over Zola. Since the site uses an off-the-shelf theme rather than hand-written templates, Hugo's far larger theme ecosystem is the deciding factor. Hugo also ships an officially maintained GitHub Pages workflow in its own docs; Zola delegates to a third-party action. |
| Theme | Blowfish v3.1.0 | MIT, ~2.9k stars, most actively developed of the candidates considered (Blowfish, PaperMod, Congo). Requires Hugo >= 0.162.0. |
| Theme install | Git submodule at `themes/blowfish`, pinned to tag `v3.1.0` | Blowfish's own docs recommend submodule over Hugo Modules. Keeps theme and content cleanly separated and updatable. |
| Old site | Deleted from the working tree | Recoverable from git history at commit `a4fa12b`. `LICENSE` retained, `README.md` rewritten. |
| Scope at launch | Posts index, post pages, tags, Atom/RSS feed | No `/about/` page. Trivial to add later. |

## Site identity

- **Title:** `Patrick Owens`
- **Tagline:** `Software, radio, and small plastic soldiers.`
- **Header/footer links:** GitHub (`https://github.com/powens`) and the RSS feed.

## Target repo layout

```
.github/workflows/hugo.yaml
.gitmodules
config/_default/hugo.toml          # theme, baseURL, taxonomies
config/_default/languages.en.toml  # title + tagline
config/_default/markup.toml        # goldmark + syntax highlighting
config/_default/menus.en.toml      # nav + GitHub/RSS links
config/_default/params.toml        # Blowfish appearance settings
content/_index.md
content/posts/_index.md
content/posts/hello-world/index.md
archetypes/default.md
themes/blowfish/                   # submodule @ v3.1.0
LICENSE
README.md
.gitignore                         # public/, resources/_gen/, .hugo_build.lock, .DS_Store
```

No `static/` directory is created at launch. Git cannot track an empty
directory, and there is no static asset to put in one yet; add it when the first
favicon or image needs it.

Blowfish expects configuration split across `config/_default/` rather than a
single `hugo.toml`. `module.toml` is deliberately omitted: it applies only to
the Hugo Modules install path, which is not being used. Because the theme is a
submodule, `theme = "blowfish"` must be set in `config/_default/hugo.toml`.

`content/posts/hello-world/index.md` must carry at least one entry in its
front-matter `tags` list, so that the taxonomy output is actually exercised by
the verification below.

Tags and feeds require no extra configuration. Hugo generates taxonomy pages and
`index.xml` from core, so front-matter `tags: [...]` is sufficient.

## Deployment

Hugo's official GitHub Pages workflow (from the Hugo docs), publishing via the
Pages artifact mechanism rather than a `gh-pages` branch, with four deliberate
modifications:

1. **Trigger on `master`.** This repo's default branch is `master`; the upstream
   sample uses `main`.
2. **`submodules: recursive` on checkout.** Required for the theme to exist at
   build time. Without it the build fails with a missing-theme error.
3. **Drop the Dart Sass install step.** Verified unnecessary: Blowfish contains
   zero `.scss` files and ships precompiled Tailwind CSS in
   `assets/css/compiled/`. Hugo extended is likewise not required.
4. **Drop `setup-go` and `setup-node`.** Go is needed only for the Hugo Modules
   install path; Node only for themes that build their own CSS. Neither applies.

Modifications 3 and 4 are the only ones carrying risk. Both are cheap to reverse
and are validated before push by the local build; if CI disagrees with the local
result, restore the removed steps rather than debugging further.

Permissions: `contents: read`, `pages: write`, `id-token: write`.
Concurrency group `pages`, `cancel-in-progress: false`.

`baseURL` is injected at build time via
`--baseURL "${{ steps.pages.outputs.base_url }}/"`, so it cannot drift out of
sync with the repository's Pages configuration.

## Manual steps (owner-performed, not automated)

Repo Settings -> Pages -> Source -> `GitHub Actions`. Required before the first
deploy can succeed. This cannot be reliably scripted and must be done by the
repository owner.

## Verification

The work is done when all of the following hold:

1. `hugo` is installed locally (`brew install hugo`).
2. `hugo build` exits zero with no errors.
3. `public/index.xml` exists (the feed).
4. `public/tags/` exists and contains a page for the sample post's tag (see
   the front-matter requirement above).
5. The GitHub Actions run completes green.
6. `https://powens.github.io` loads, showing the sample post, a working tag
   page, and functioning GitHub and RSS links.

Steps 5 and 6 depend on the owner completing manual step 1 first.

## Out of scope

- An `/about/` page.
- Any custom domain. The site lives at `https://powens.github.io` and no DNS is
  touched.
- Preserving the old Tau page at a sub-URL. It lives in git history only.
