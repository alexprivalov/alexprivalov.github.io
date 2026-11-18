# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Personal Jekyll blog (mostly Russian; travel reports and C++/Linux notes), served by GitHub Pages from `master`. Based on the Poole theme. No tests, no lint, no CI — pushing to `master` deploys.

## Commands

Preferred: Docker (no local Ruby needed). Gems are cached in the `blog-gems` volume; `--force_polling` makes regeneration work on macOS bind mounts.

```sh
docker run -d --name blog -v "$PWD":/srv/jekyll -v blog-gems:/usr/local/bundle -w /srv/jekyll -p 4000:4000 \
  ruby:3.3 bash -c 'bundle install && bundle exec jekyll serve --host 0.0.0.0 --drafts --force_polling'
docker logs -f blog    # stop with: docker rm -f blog
```

Without Docker:

```sh
bundle install                      # github-pages gem pins Jekyll/plugin versions to what GH Pages runs
bundle exec jekyll serve            # preview at http://localhost:4000
bundle exec jekyll serve --drafts   # also render _drafts/
bundle exec jekyll build            # one-off build into _site/ (gitignored)
```

`future: false` in `_config.yml`: posts dated in the future are not built unless you pass `--future`.

## Structure that matters

- `_posts/YYYY-MM-DD-slug.md` — every post sets an explicit `permalink:` in front matter (e.g. `permalink: docker-cpp`), so URLs are `/<permalink>`, not date-based. Keep existing permalinks stable; other posts link to them.
- Front matter pattern: `layout: post`, `title`, `permalink`, `tags` (space-separated; `tags.html` builds the tag index from them).
- `_drafts/` — unpublished posts (no date prefix required).
- `assets/<post-slug>/` — images per post, referenced as absolute paths (`/assets/docker-c++/...`). Posts embed images with raw `<img ... style="width: 720px;">` rather than Markdown image syntax.
- `public/` — theme CSS/JS/icons (`poole.css`, `syntax.css`, `scrollup.js`).
- `_layouts/default.html` → `post.html` / `page.html`; `_includes/head.html` is the only include. Header nav comes from `pages_list` in `_config.yml`.
- Plugins (must be on the GitHub Pages whitelist): `jemoji`, `jekyll-paginate`, `jekyll-gist`. Pagination applies to `index.html`.
- Translations live as separate posts with a `-en` suffix (e.g. `2018-03-09-docker-cpp-en.md`, `permalink: docker-cpp-en`) and link back to the original.

## Known issues

Older travel posts (2015–2018) embed GPSies map iframes (`gpsies.com/mapOnly.do?fileId=...`); GPSies is defunct, so those embeds are broken. No GPX files are stored in the repo.
