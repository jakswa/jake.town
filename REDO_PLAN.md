# REDO PLAN — jake.town visual refresh

Handoff doc for a fresh agent session. Read top to bottom, then start at "Step 0".

## Mission

The owner is tired of this personal website's look/feel and wants a full visual
refresh. His words:

> "complete creative freedom -- new styles/fonts/look/feel. keep the resume
> content (work history/etc) and blog post content (title/body) unchanged.
> I cannot say for sure what I will dislike about your results, but I dislike
> the current site's busyness/noise -- think I would've let it sit longer if
> it was somewhat more minimal."

## What went wrong with the previous attempt (do not repeat)

A prior attempt redesigned the layouts but **reused the same two fonts that
were already on the site** (self-hosted Newsreader + JetBrains Mono in
`assets/fonts/`). The owner called this out — with "complete creative freedom,"
keeping the fonts is not freedom, it's path-of-least-resistance. **Pick genuinely
new fonts.** That is the single most important correction for this attempt.

Second, smaller lesson: the owner is a picky, opinionated iterator ("I cannot
say for sure what I will dislike"). After building, present the result briefly
and expect a feedback round. Don't over-engineer before showing him.

## Step 0 — reset to a clean tree

The working tree from the failed attempt is mid-edit and broken (two layouts
deleted, others rewritten inconsistently). Throw it all away first:

```
git checkout -- .        # restores _layouts/*, index.html, etc.
# (untracked mise.toml and the Gemfile.lock bump are environment noise — leave or revert, irrelevant)
```

Sanity: `git status` clean, `_layouts/` has `default.html home.html post.html
resume.html` again, `bundle exec jekyll build` succeeds.

The pre-redesign site (the "terminal / phone-operator" theme) is always
retrievable via `git show HEAD:<path>` — e.g. `git show HEAD:_layouts/default.html`
if you want to lift working pieces (giscus config, OG meta, print rules).

## Build setup

- Jekyll 4.3 (`Gemfile`: jekyll, webrick). Ruby 3.4.9 (`.ruby-version`, `mise.toml`).
- Dev: `bundle install && bundle exec jekyll serve` → http://localhost:4000
- `_site/` and `.jekyll-cache/` are gitignored build output — never edit them.
- No external build steps, no JS framework. It's a static Jekyll site for GitHub Pages.

## Site architecture

| File | Role |
|---|---|
| `_config.yml` | `markdown: kramdown`, `permalink: /blog/:slug/`, site title/description/url |
| `_layouts/default.html` | Shared chrome: `<head>` (fonts, OG/Twitter meta, base CSS) + header + footer. Used by blog index + (old) post. |
| `_layouts/home.html` | Standalone doc for `index.html` (old: full-viewport "operator console"). |
| `_layouts/post.html` | Old: chains to default, adds article styles + giscus comments. |
| `_layouts/resume.html` | Standalone doc for `resume.html` (old: terminal "session" theme). |
| `index.html` | Home. |
| `blog/index.html` | Post index (had a tag filter — **posts have no tags, it never rendered**; safe to drop). |
| `resume.html` | The résumé. |
| `_posts/*.md` | 9 posts. **Do not edit.** |
| `assets/fonts/*.woff2` | Old self-hosted fonts (Newsreader x2, JetBrains Mono). Replace. |
| `assets/images/*` | Post images (webp/gif). Keep as-is. |
| `CNAME`, `llms.txt`, `README.md` | Leave alone. |

Free rein over: all layouts, all page files, `_config.yml`, font files, CSS.

## Invariants — content that must survive (verbatim)

### 1. Blog posts: don't touch the files

`_posts/*.md` is off-limits. Their frontmatter drives rendering — the new post
layout must keep working with: `title`, `slug`, `date`, `description` (lede/
excerpt), `image` (only 2 posts have it — OG image), `layout: post`.

Post bodies contain raw HTML the new article styles must still handle:
`<figure>`, `<img class="hero-image">`, `<figure class="hero-image">`,
`<img class="float-left">`, `<figure class="float-left">`, `<table>`s, kramdown
fenced code blocks (`<pre><code>`). Check `git show HEAD:_posts/...` per file
if unsure.

### 2. Resume content: same facts, new presentation

Source of truth for exact wording: `git show HEAD:resume.html`. Everything
factual must survive with identical wording:

- Identity: Jake Swanson — "Staff Engineer · backend systems, some frontend,
  lots of phone calls." · Atlanta, GA · (678) 719-9096 · jakswa@gmail.com ·
  linkedin.com/in/jakswa · github.com/jakswa
- **Experience** (titles, companies, date ranges, per-job stack lists, and
  every bullet — word for word):
  - Staff Engineer @ CallRail, Dec 2024 → Present
  - Staff Engineer @ Grayscale Labs, Oct 2022 → Dec 2024
  - Staff Engineer @ CallRail, Oct 2014 → Aug 2022
- **Stacks** section: 5 keys with exact values (voice_realtime, ai_workflow,
  languages, frameworks, infrastructure).
- **Projects**: kincall.app, marta.io, home.jake.town — with their exact
  description lines and URLs.
- **Writing**: links to 4 posts (exact post titles).
- **Education**: Georgia Southern University, B.S. in Computer Science, 2009 – 2011.
- **A commented-out block** in resume.html (a "Hummingbird" / founding-engineer
  TODO) — keep it as an HTML comment in the new file too; he may want it later.

Decorative text from the old theme is NOT content and can die: "ON SHIFT",
"uptime 12y 7mo", "load 0.42", "session_1", "● CONNECTED", "connection closed.
have a good shift.", `jake --resume --format=verbose`, etc.

### 3. Functionality to keep

- Giscus comments on posts (config in old `post.html`: repo `jakswa/jake.town`,
  repo-id `R_kgDOQzmm9A`, category `General` / `DIC_kwDOQzmm9M4C0kwg`,
  mapping `pathname`, lazy load). Pick a light-theme variant.
- Open Graph + Twitter card meta on all pages (og:image from `image:`
  frontmatter on posts).
- `tel:` and `mailto:` links.
- RSS: the old blog page links `/feed.xml`, **but no feed is generated** (it 404s
  today). Either add the `jekyll-feed` gem (needs `bundle install`, then it
  appears in `_site/feed.xml`) or drop the link. Adding the gem is the better
  fix.
- Decent print output (the old site had print styles; a résumé especially
  benefits).
- Mobile-friendly, no horizontal scroll.
- Favicon: old one is a 📞 in a data-URI — keep or replace, his call.

## Gotchas / context from exploration

- The old design is a deliberate "phone operator terminal" theme: dark bg,
  scanlines, oscilloscope, dial pad, blinking pips, uppercase micro-labels,
  fake terminal chrome. That *is* the noise he's tired of. The refresh should
  move away from it, not restyle it.
- Old fonts live in `assets/fonts/` as Latin-subset woff2 files, self-hosted
  with `@font-face` + `<link rel="preload">`. That self-hosting pattern is good
  (fast, no third-party requests, works offline) — **keep the pattern, change
  the fonts.**
- The old post layout used CSS custom properties (`--glow`, `--paper`, etc.)
  defined per-layout; there was no single design system. A single source of
  truth for styles (one shared layout or one CSS file) is an improvement.

## Design direction (strong steer, final call is the session's)

- **Minimal and quiet.** "Somewhat more minimal" is the explicit ask. Plenty of
  whitespace, one column, low-contrast supporting text, at most one accent
  color, nothing that moves or blinks, no fake chrome/boxes/badges. Test:
  would the owner be happy to never look at it again? That's the bar.
- **New fonts, obviously new.** Nothing visually close to Newsreader or
  JetBrains Mono. Self-host the new ones as `assets/fonts/*.woff2` (replace the
  old files; delete unused ones). Reliable source for ready-made Latin-subset
  woff2: the Fontsource GitHub repo —
  `github.com/fontsource/fontsource` → `files/<family>/files/<family>-latin-<weight>-<style>.woff2`
  (or the npm tarballs of `@fontsource/<family>`). Grab the weights you actually
  use (e.g. 400/500 for body + one italic if used).
- Some plausible pairings, if they help (or pick entirely different ones):
  - Characterful display serif + neutral sans: **Fraunces** (headings) +
    **Public Sans** or **Inter** (UI) — or
  - Refined editorial all-serif body with a mono for tiny meta: **Spectral** or
    **Source Serif 4** + **IBM Plex Mono** (small doses only — dates, labels) — or
  - Clean grotesk everything: **Schibsted Grotesk** or **Inter Tight**.
- Whichever way, keep the type system small: one body face, one optional accent
  face, a small mono for dates/labels is the most the old site ever needed.

## Definition of done

1. `bundle exec jekyll build` succeeds; all
   routes render: `/`, `/blog/`, `/blog/<slug>/` for all 9 posts, `/resume`,
   `/feed.xml` (if adding the gem).
2. `git diff -- _posts` is empty.
3. Every word of the résumé's factual content (section 2 above) appears in the
   new `resume.html`, and the commented-out Hummingbird block is preserved.
4. New fonts load (check the built page references them; old font files gone
   from `assets/fonts/` unless reused on purpose).
5. No console-breaking JS; giscus script intact; OG meta present on a post.
6. Looks good at ~375px and ~1280px; print view of `/resume` is sane.
7. Committed with trailer `Co-authored-by: pi <pi@local>`.

Then: show the owner and iterate.
