# powens.github.io

Hugo + Blowfish personal blog, deployed to GitHub Pages. See README.md for setup.

## Commands

```bash
hugo server -D                              # dev server with drafts, http://localhost:1313
hugo build --gc --minify                    # same build CI runs; use to verify changes
hugo new content posts/<slug>/index.md      # new post (page bundle)
git submodule update --init --recursive     # if themes/blowfish is empty
```

Pushing to `main` deploys via `.github/workflows/hugo.yaml` (pins Hugo 0.165.0; bump
`HUGO_VERSION` there if a change needs a newer Hugo).

## Where things go

- `themes/blowfish/` is a pinned submodule (v3.1.0). **Never edit it.** Override instead:
  - templates → `layouts/partials/` (same path as the theme file)
  - styles → `assets/css/custom.css` (loaded last)
  - UI strings → `i18n/en.yaml` (merged over the theme's; add only new/changed keys)
- Site config is split across `config/_default/*.toml`.
- `public/`, `resources/_gen/` are build output; `docs/superpowers/` is local scratch. All gitignored.

## Posts

- Page bundles: `content/posts/<slug>/index.md`, images alongside, referenced by bare filename.
- TOML front matter (`+++`): `title`, `date` (with `-07:00`/`-08:00` offset), `draft`, `tags`,
  `description`; optional `summary`.
- Blowfish shortcodes are available, e.g. `{{< alert "circle-info" >}}`.

## Gotchas

- `config/_default/markup.toml` must keep the `goldmark.extensions.passthrough` block: Hugo
  replaces (not merges) the theme's markup config, and without it KaTeX math breaks.
- Overrides carry a comment explaining why they deviate from the theme (often accessibility/
  contrast reasoning). Keep that convention and read it before changing an override.
