# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The Hugo source for jamiecraane.dev, a personal blog about fullstack software development (Kotlin, KMM, Ktor, AI-assisted development). There is no application code, no test suite, and no CI: the deliverable is markdown in `content/post/` plus the generated site in `docs/`.

## Commands

```bash
hugo server -D     # local preview at localhost:1313, includes drafts
hugo               # production build into docs/
hugo -D            # production build including drafts
```

`hugo` is pinned to 0.158.0 in `.tool-versions` and provided by a version manager (asdf/mise), so it is not on the PATH of a bare non-login shell. If `hugo: command not found`, the shim environment is not loaded.

## Publishing model

`publishDir = "docs/"` in `config.toml`, and `docs/` is committed to git. GitHub Pages serves that directory on `main`; `static/CNAME` maps it to jamiecraane.dev. A content change is therefore two things in one commit: the markdown under `content/` and the regenerated HTML under `docs/`.

**Before committing `docs/`, always run a production `hugo` build and confirm the output is clean.** `hugo server` writes a development build into the same directory, containing `localhost:1313` URLs and injected livereload scripts. Committing that breaks the live site. Verify with:

```bash
grep -rl "localhost:1313" docs/ ; grep -rl "livereload" docs/
```

Both should return nothing. Kill any running `hugo server` before the final build.

Removing or renaming a post leaves orphaned pages in `docs/` (post page, tag pages, category pages). Hugo does not clean them up, so delete them explicitly and check `docs/sitemap.xml`, `docs/index.xml`, and `docs/algolia.json`.

## Post conventions

Writing style, frontmatter template, structure and header-image guidance live in [SKILL.md](air-file://cj2kp5g1rple18tqrlcf/Users/jamiecraane/develop/IntelliJ/PriveProjecten/personalblog/.claude/skills/write-blog-post/SKILL.md?type=file&root=%252F) (the `write-blog-post` skill). Use it for any drafting or editing work. The load-bearing details:

- Filename is `YYYY-DD-MM-<slug>.md` and the `URL` frontmatter field is `/YYYY/DD/MM/<slug>/`, both in **day-month order**. The `date` field is ordinary `YYYY-MM-DD`. Older posts predate this convention and use `YYYY-MM-DD` filenames, so match the recent ones rather than the neighbours in the directory listing.
- `URL` is explicit on every post, so the on-disk path does not determine the published path. Changing it orphans the old `docs/` directory.
- Header images go in `static/img/posts/<slug>.jpg` and are referenced as `/img/posts/<slug>.jpg`. They are rendered as a CSS `background-image` on the post header, so JPEG at roughly 1600px wide, quality 82, not PNG. Older posts still reference PNGs and images directly under `/img/`.
- No em dashes anywhere in prose, including headings. Use commas, colons, or parentheses.

`content/_wip/` and `content/top/` are both built and published; the underscore is not a Hugo exclusion here.

## Theme and layout overrides

The theme `hugo-theme-cleanwhite` is vendored under `themes/` and tracked in git, not a submodule. `layouts/` holds only project-level overrides of theme files (currently `_default/page.html` and `partials/comments.html`). To change rendering, copy the theme file to the matching path under `layouts/` and edit the copy rather than editing the theme in place.

The home page emits an extra `Algolia` output format (`docs/algolia.json`) alongside HTML and RSS, defined by `[outputFormats.Algolia]` in `config.toml`. Search is currently commented out in the config, so the index is generated but unused.
