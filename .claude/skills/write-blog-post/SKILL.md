---
name: write-blog-post
description: Write or edit a blog post for jamiecraane.dev in Jamie's style. Use when drafting a new post, shortening/rewriting a draft, or adding a post to the site (Hugo, content/post/, published from docs/).
---

# Writing a blog post in Jamie's style

This is a Hugo site. Posts live in `content/post/`, the site is published from `docs/` (GitHub Pages), and `hugo build` regenerates it.

## Workflow

1. Create the post in `content/post/` using the filename and frontmatter conventions below.
2. If a header image exists, place it in `static/img/posts/<slug>.png` and reference it in the frontmatter. If not, offer an image-generation prompt (see Header images).
3. Run `hugo build` and verify the post appears under `docs/<URL path>/index.html`.
4. Do not commit unless asked. The regenerated `docs/` files are part of the change and get committed together with the post.

## Filename and URL

- Filename: `YYYY-DD-MM-<slug>.md` (day-month order, matching the most recent posts, e.g. `2026-30-04-reliability-engineering-in-ai-era.md`).
- `URL` frontmatter field: `/YYYY/DD/MM/<slug>/` (same day-month order).
- The `date` field itself is normal `YYYY-MM-DD`.
- Slug: short, lowercase, hyphenated.

## Frontmatter template

```yaml
---
layout: post

title: "Title In Title Case"
subtitle: "One-line elaboration of the title"
author: Jamie Craane
date: YYYY-MM-DD
description: "One or two sentences for SEO/preview, written as a plain summary."
image: "/img/posts/<slug>.png"
showtoc: false
tags:
- AI
- Kotlin

categories: [ AI ]
URL: "/YYYY/DD/MM/<slug>/"
---
```

- Tags are capitalized, human-readable phrases (`AI-Assisted Development`, `Claude Code`, `Ktor`), one per line.
- `categories` is one or two broad buckets: `[ AI ]`, `[ Kotlin ]`, `[ Reliability ]`.
- `showtoc: true` only for long reference-style posts with many sections.

## Voice and style

- First person, experience-driven. Posts open from a concrete situation ("Recently I had exactly this situation"), not from abstract context.
- Practical and direct. Opinions are stated plainly ("My take? The operator carries the responsibility.").
- Short paragraphs, two to four sentences. No filler or throat-clearing.
- **Never use em dashes (—).** Use commas, colons, or parentheses instead. This includes headings: write `Stage 1: Visualize`, not `Stage 1 — Visualize`.
- Bold the key phrase of a point so the post scans well, especially as lead-ins to list items: `**Verify the parser.** An AI-generated parser can...`.
- Use italics for emphasis on single words (*this* log, *any* file).
- Bulleted lists and numbered lists are used freely; prefer a list over a paragraph enumerating three or more things.
- Always include at least one concrete example: a code block, log excerpt, config snippet, or prompt. Fenced blocks with a language tag (`kotlin`, `text`, plain for logs).
- Be honest about limitations: recent posts include a short caveats section ("A few caveats keep it honest") or qualify recommendations ("treat it as a maturity ladder rather than a checkbox").

## Structure

Typical shape (top-level sections use `###`, subsections `####`; some older posts use `##`, either is fine but be consistent within a post):

1. `### Introduction`: the concrete situation and the questions it raises (often as a bulleted list).
2. The problem: why the obvious approaches fall short.
3. The solution: what was actually done, with examples.
4. Deeper exploration: stages, examples, or best practices as subsections.
5. Caveats (optional but preferred for opinion/AI posts).
6. `### Conclusion`: tie back to the opening, end with a forward-looking thought.
7. `### References` (optional): bulleted markdown links, only when external sources were genuinely used.

Target length: roughly 800-1200 words for an opinion/workflow post. Tutorial posts (Ktor, Firebase) can be longer and code-heavy.

## Header images

Existing post images are dark conceptual tech illustrations: deep slate-blue palette, glowing translucent UI elements, subtle purple/amber accents, isometric depth, landscape (around 1792x1024 or wider). When proposing an image-generation prompt:

- Describe the post's core concept visually (transformation, flow, layers).
- Always include "no readable text" (image models garble text).
- Specify wide landscape composition.
- Save the result as `static/img/posts/<slug>.png`.
