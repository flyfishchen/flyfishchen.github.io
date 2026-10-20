# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal blog of flyfishchen, built with Astro (blog starter template) and deployed to GitHub Pages at https://flyfishchen.github.io. Site content is written in Chinese (`<html lang="zh-CN">`).

## Commands

Requires Node >= 22.22.

- `npm install` — install dependencies
- `npm run dev` — dev server at http://localhost:4321. When starting it from Claude Code, run it in the background: `npx astro dev --background`, managed with `npx astro dev stop|status|logs`
- `npm run build` — static build into `dist/`; this also type-checks post frontmatter against the content schema, so use it to verify changes (there are no tests or linter)
- `npm run preview` — serve the built `dist/`

## Architecture

- **Posts** live in `src/content/blog/*.md|mdx`. The filename becomes the URL slug (`/blog/<slug>/`). Frontmatter is validated by the schema in `src/content.config.ts`: `title`, `description`, `pubDate` are required; `updatedDate` and `heroImage` (relative path to an image in `src/assets/`) are optional. The layout renders `title` as the page `<h1>`, so post bodies should not repeat the title as a `#` heading.
- **Routing**: `src/pages/blog/[...slug].astro` renders each post via `src/layouts/BlogPost.astro`; the home page (latest 10) and `src/pages/blog/index.astro` (all) both render posts through `src/components/PostList.astro`; `src/pages/rss.xml.js` builds the RSS feed. `about.astro` reuses the `BlogPost` layout without a `pubDate`.
- **Styling** is deliberately minimal: a single narrow column, system CJK font stack, no web fonts. All colors and widths are CSS variables at the top of `src/styles/global.css`, redefined under `prefers-color-scheme: dark`; components use those variables rather than hard-coded colors. Code highlighting uses Shiki dual themes (`github-light`/`github-dark`, set in `astro.config.mjs`), switched to dark by a rule at the end of `global.css`. Markdown tables are `display: block` with `nowrap` cells so wide tables scroll horizontally.
- **Site-wide values**: title/description in `src/consts.ts`; `site` URL in `astro.config.mjs` (used for RSS, sitemap and canonical URLs). `og:image` is only emitted when a page has a `heroImage`.
- **Static files** in `public/` are copied verbatim to the site root. `public/.nojekyll` and `public/10e7a4090fb0ad8f0a2ca9de952e4b74.txt` (a site verification file) must stay there.

## Writing posts

Readers are mostly on phones and many don't care about details. Structure every post as conclusion first, evidence after:

- Open with a `## 先说结论` section of 3–5 short bullets, each a complete claim a reader can stop after. Follow with a short actionable section (e.g. `## 怎么做`) when the post leads to a recommendation.
- Then a horizontal rule and one line telling readers the rest is the argument and can be skipped; data tables, derivations, configs and caveats go below it.
- Keep paragraphs to 2–3 sentences. Prefer bullets and short per-item lists over wide tables near the top; wide tables are fine in the evidence part (they scroll horizontally).
- Mark what is measured, what is from official docs, and what is speculation (`（推测）`). When a later test contradicts an earlier claim, rewrite it to the current finding; don't keep correction notes or the old wording.
- Cite sources at the end under `## 参考资料`, and set `updatedDate` in frontmatter when revising a published post.
- Don't name private infrastructure (internal gateways, hosts, account or project names).

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml` (`withastro/action` → `actions/deploy-pages`). The repo's Pages source must be set to "GitHub Actions". The repo is private, so git over HTTPS authenticates through the `gh` credential helper.
