# AGENTS.md

Guide for coding agents working on this repository.

## Overview

Personal blog of Filippo Buletto, published at <https://blog.filippobuletto.info>.

- Static site built with Jekyll 3.x (`Gemfile`: `jekyll ~> 3.5`) and kramdown with the GFM parser.
- Theme: [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) `4.28.1`, installed as the `minimal-mistakes-jekyll` gem with the `dark` skin. Theme files are not vendored in this repo.
- Hosted on Cloudflare Pages (build system v3): `_headers` sets the HTTP response headers and `.ruby-version` sets the build Ruby version.
- Comments use giscus (GitHub Discussions on `filippobuletto/blog`). The contact button uses Formspree.
- Content is licensed CC BY 4.0 (`LICENSE`).

## Layout

- `_config.yml`: site settings, plugins, front matter defaults, collections, giscus config.
- `_posts/`: blog posts, named `YYYY-MM-DD-slug.md`.
- `_extrapolation/`: "Extrapolation" collection, one entry per month (`YYYY-MM.md`), served at `/extrapolation/:path/`.
- `_pages/`: standalone pages (about, terms, archives, 404, extrapolation index, and others). Each page sets its own `permalink`.
- `_data/navigation.yml`: top navigation (`main`) and the `kubernetes-toolbox` sidebar.
- `_includes/`: theme overrides only.
  - `head/custom.html`: Iosevka webfonts and favicons.
  - `footer/custom.html`: terms link and Formspree contact button.
  - `comments-providers/giscus.html`: giscus loader.
  - `author-profile-custom-links.html`.
- `assets/css/main.scss`: font variables, then imports the theme skin and partials.
- `assets/images/`: post images. `assets/docs/`: PDFs (theses). `assets/filippo.buletto.pubkey.asc`: PGP key.
- `.well-known/webfinger`: Mastodon identity (`acct:filippomito@mastodon.uno`).
- Root icons (`favicon*`, `android-chrome-*`, `mstile-*`, `site.webmanifest`, `browserconfig.xml`): favicon set.
- `index.html`: home page (`layout: home`).

## Local development

There is no `Gemfile.lock` (it is gitignored), no CI, and no test suite. The only check is a clean build.

`.ruby-version` pins Ruby to 3.4.4. Cloudflare Pages (build system v3) reads this file to choose the Ruby version, so use the same version locally. Change it only after `bundle exec jekyll build` passes on the new version.

Always run Jekyll through `bundle exec`. The Cloudflare v3 build image ships a global Jekyll 4.x that conflicts with the `Gemfile` versions. The Cloudflare Pages build command, set in the dashboard, must be `bundle exec jekyll build` with output directory `_site`. A plain `jekyll build` fails with `Gem::LoadError`.

```sh
bundle install
bundle exec jekyll serve --livereload   # http://localhost:4000
bundle exec jekyll build                # output in _site/
```

Run `bundle exec jekyll build` after every change and fix any errors or warnings it reports.

## Writing a post

1. Create `_posts/YYYY-MM-DD-slug.md`.
2. Use this front matter, modelled on `_posts/2026-08-07-nut-ups-monitoring.md`:

   ```yaml
   ---
   title: "Post title"
   excerpt: "One-sentence summary shown in lists and previews."
   permalink: /slug/
   categories:
     - Programming
   tags:
     - java
   header:
     overlay_image: /assets/images/slug.jpg
     overlay_filter: rgba(0, 0, 0, 0.5)
   toc: true
   toc_sticky: true
   toc_icon: "cog"
   ---
   ```

   - `title`, `excerpt`, `permalink`, `categories`, and `tags` appear in every post.
   - `header`, `toc`, `toc_sticky`, and `toc_icon` are optional.
   - `permalink` is always `/slug/` at the root. Never change the permalink of a published post.
   - Reuse an existing category: `Programming`, `Thoughts`, `DevOps`, `Sysadmin`, `Fit Life`, `monitoring`.
   - Don't repeat values that `_config.yml` `defaults` already sets: `layout: single`, `author_profile`, `read_time`, `show_date`, `share`, `related`, `comments`.
   - giscus maps discussions by `og:title`. Renaming a published post's `title` detaches its comments.
3. Put images in `assets/images/` and reference them with absolute paths (`/assets/images/name.jpg`). External image URLs also work.
4. Style blocks with the kramdown attribute lists already used in posts:

   ```markdown
   Important note.
   {: .notice--info}

   ![alt](/assets/images/name.jpg)
   {: style="text-align: center;"}

   [Download](/assets/docs/file.pdf){: .btn .btn--success}
   ```

   The available notice classes are `.notice`, `--primary`, `--info`, `--warning`, `--success`, and `--danger`.

## Other content

- Extrapolation entry: `_extrapolation/YYYY-MM.md` with `title` (format `"YYYY-MM: Topic"`), `excerpt`, and optionally `toc` and `classes: wide`. `_pages/extrapolation.md` lists the entries automatically.
- New page: add `_pages/name.md` with `title` and `permalink`. Add a `main` item in `_data/navigation.yml` if the page belongs in the top nav.
- Gists: `jekyll-gist` is enabled (`{% gist id %}`). Emoji: `jemoji` (`:smile:`).

## Theme customization

- Customize the theme only through the override points under `_includes/` and through `assets/css/main.scss`. Don't copy whole theme layouts into the repo unless an override hook doesn't exist.
- Check the theme docs for version 4.28.1 before you use any front matter or config option: <https://github.com/mmistakes/minimal-mistakes/tree/4.28.1/docs/_docs>
- Enabled plugins must appear in both `plugins` and `whitelist` in `_config.yml`, and in the `jekyll_plugins` group in `Gemfile`.

## Security headers

`_headers` defines a Content Security Policy. Scripts are allowed from `'self'`, `*.filippobuletto.info`, `giscus.app`, and `formspree.io`. Frames are allowed from `'self'`, `giscus.app`, and `formspree.io`. Any new external script, iframe, or embed needs a matching CSP update in `_headers`, or the browser blocks it in production.

## Git

- Work on `develop`. `master` is the main branch.
- Commit messages follow Conventional Commits, e.g. `feat(post): add UPS monitoring post`, `fix(nav): correct toolbox link`.
- Don't commit build output or local state: `_site/`, `.jekyll-cache/`, `.sass-cache/`, `Gemfile.lock`, `vendor/`.

## Avoid

- Editing favicons, `.well-known/webfinger`, the PGP key, or the thesis PDFs unless the task asks for it.
- Changing the giscus `repo_id`/`category_id` or the Formspree endpoint.
- Changing permalinks or the `permalink` pattern in `_config.yml`: both break existing URLs and comments.
