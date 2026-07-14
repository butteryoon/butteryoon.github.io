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
- Reference site assets as `{{site.baseurl}}/assets/img/...`.

## Structure

- `_layouts/` (default → main/page/post) and `_includes/` (head, header, footer, analytics, Disqus comments, Google AdSense, MS Clarity) make up the theme.
- Styles: `_sass/` compiled via Jekyll; syntax highlighting is Rouge.
- Pagination: 10 posts/page at `/page/:num`; `tags.html` renders the tag index.
- Root-level `google*.html`, `naver*.html`, `BingSiteAuth.xml`, `ads.txt` are search-engine/ads verification files — do not modify or delete.
