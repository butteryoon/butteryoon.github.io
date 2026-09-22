# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Korean-language tech blog (butteryoon.github.io) built with Jekyll on the Flexible-Jekyll theme, hosted on GitHub Pages (`master` branch is the published site). Posts cover Windows/Linux terminal topics; content is written in Korean.

## Commands

- `bundle exec jekyll serve` — build and serve locally with live reload (Windows: `wdm` gem handles file watching).
- `bundle exec jekyll build` — build to `_site/`.
- `bundle exec jekyll serve --drafts` — include `_drafts/` posts.

Publishing = commit and push to `master`; GitHub Pages builds automatically. Plugins used (sitemap, paginate, gist, jemoji, jekyll-youtube) are all GitHub Pages–whitelisted.

## Writing posts

- Posts live in `_posts/YYYY-MM-DD-title.md`; drafts in `_drafts/` (which also holds stray images — post images normally go in `assets/img/`).
- Use `_drafts/0000-00-00-templete.md` as the front matter template. Required front matter: `layout: post`, `comments: true`, `title`, `description`, `img` (thumbnail in `assets/img/`), `date` and `last_modified_at` (with `+0900` timezone), `tags`, `related`, `categories`.
- Summaries are cut at `<!--more-->` (`excerpt_separator`) — every post should include it after the intro paragraph.
- Update `last_modified_at` when editing an existing post (recent commits show this convention).
- `date` must not be in the future: GitHub Pages excludes future-dated posts from the build (no `future: true` in config), so the post silently won't appear on the site. When writing a new post, set `date` to the current time or earlier (KST, `+0900`).
- `_config.yml` sets `timezone: Asia/Seoul` so post URLs use KST dates. Do not remove it — without it GitHub Pages builds in UTC and posts published before 09:00 KST get URLs dated one day earlier, breaking internal links.
- Reference site assets as `{{site.baseurl}}/assets/img/...`.
- **Every post needs a well-formed `last_modified_at`** — it is the sort key for the home listing (see Structure). Use exactly `YYYY-MM-DD HH:MM:SS +0900`; a malformed value (e.g. one-digit seconds `18:00:0`) is parsed as a String instead of a Time by Ruby's YAML, and the mixed types make the GitHub Pages build fail. Validate with `^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2} [+-]\d{4}$`.
- **Posts use two extensions: `.md` and `.markdown`.** When auditing or bulk-editing front matter, glob `_posts/*`, never `_posts/*.md` — 7 older posts are `.markdown` and are silently skipped otherwise.

## Structure

- `_layouts/` (default → main/page/post) and `_includes/` (head, header, footer, analytics, Disqus comments, Google AdSense, MS Clarity) make up the theme.
- Styles: `_sass/` compiled via Jekyll; syntax highlighting is Rouge.
- Pagination: 10 posts/page at `/page/:num`; `tags.html` renders the tag index.
- **Home listing is ordered by `last_modified_at`, not `date`.** `index.html` sorts all of `site.posts` by that key and then slices the current page's range, so refreshed old posts resurface at the top. Do not sort inside `paginator.posts` — that only reorders within a page and breaks the page boundaries. Post URLs, `sitemap.xml`, and category/tag indexes stay `date`-based and must not change.
- **Local build success ≠ GitHub Pages success.** Local runs Jekyll 4.x; Pages runs Jekyll 3.10 + Liquid 4.0.4, whose `sort` filter raises `comparison of Array with Array failed` on nil or mixed-type values that 4.x tolerates. After changing Liquid in layouts or listing pages, check `gh run list` and, on failure, `gh run view <id> --log-failed | grep -i "liquid exception"`.
- Root-level `google*.html`, `naver*.html`, `BingSiteAuth.xml`, `ads.txt` are search-engine/ads verification files — do not modify or delete.
