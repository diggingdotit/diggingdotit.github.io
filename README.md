# digging.it

Source repository for [digging.it](https://diggingdotit.github.io/) — a development diary, engineering notes, and practical references built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Repository Layout

- `content/diary/` — Development diary entries and takeaways
- `content/reference/` — Practical technical references
- `content/articles/` — Longer-form write-ups
- `content/projects/` — Project pages
- `hugo.toml` — Site configuration, menus, and permalinks
- `.markdownlint.json` — Shared Markdown Lint rules (used by VS Code and GitHub Actions)
- `.github/workflows/pages.yml` — Lint, build, and GitHub Pages deployment pipeline
- `.github/dependabot.yml` — Weekly grouped updates for SHA-pinned GitHub Actions

## Publishing Workflow

1. Draft on the `development` branch with `draft: true` in frontmatter and preview locally with `hugo server -D`.
2. Check formatting locally with the VS Code Markdown Lint extension (zero local CLI dependencies).
3. Once scrubbed and ready, set `draft: false` and push `development` — GitHub Actions runs `markdownlint-cli2` over `README.md` and `content/**/*.md` without deploying.
4. Fast-forward merge `development` into `main` and push to run the full `lint` → `build` → `deploy` pipeline to GitHub Pages.

## CI & Security

- All GitHub Actions in `.github/workflows/pages.yml` are pinned to immutable 40-character commit SHAs and kept current via Dependabot.
- `lint`, `build`, and `deploy` run as separate jobs so `pages: write` and `id-token: write` permissions are scoped strictly to the deployment step.

## Local Preview

Clone with submodules so the PaperMod theme is included:

```bash
git clone --recurse-submodules https://github.com/diggingdotit/diggingdotit.github.io.git
cd diggingdotit.github.io
```

Start a live-reloading local server (including drafts):

```bash
hugo server -D
```

Preview at `http://localhost:1313/`. To run a production build locally:

```bash
hugo --minify
```
