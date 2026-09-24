# blog

Source for [psykosoldier.github.io/blog](https://psykosoldier.github.io/blog/) —
a blog about running a smart home with an AI coding agent acting as project
lead. Built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to
GitHub Pages via GitHub Actions on every push to `main`.

## How this repo fits together

New articles are drafted and reviewed in a separate, private repo first (kept
apart from this one on purpose, so nothing half-finished ends up here).
Once an article is ready, it's ported into this repo as a Hugo content page
bundle under `content/posts/<slug>/`, shipped with `draft: true`, and only
goes live once that's flipped to `draft: false` and pushed — the build
simply doesn't include draft pages, so nothing is reachable before that.

## Structure

- `content/posts/<slug>/index.md` — one page bundle per article, header image
  alongside it
- `hugo.toml` — site config, theme params, taxonomies (categories/tags)
- `themes/PaperMod/` — vendored theme (MIT license, not a submodule)
- `.github/workflows/hugo.yml` — build + deploy to GitHub Pages

## Local preview

```sh
hugo server --buildDrafts
```

## License

Content (text and images) is licensed [CC BY 4.0](LICENSE) — free to share
and adapt with attribution. The vendored PaperMod theme keeps its own MIT
license (see `themes/PaperMod/LICENSE`).

## Found a mistake?

Please [open an issue](../../issues/new/choose) — corrections and factual
errors are genuinely welcome. This isn't set up to take code contributions
(it's a single-author blog, not a collaborative software project), so there's
no separate contributing guide.
