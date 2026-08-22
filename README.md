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
