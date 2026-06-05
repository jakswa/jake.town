---
layout: post
title: "My Non-technical PM Commits More Code"
slug: pm-commits-more
description: "Two months into a spun-out, AI-heavy team, the PM who'd never written code has out-committed me. Field notes from playing Engineering Support to a very fast machine."
---

My English-major product manager had never written code before, and barely knows what a terminal is. We have been in the same git repo since early April. Here’s the commit counts:

```bash
$ git shortlog -sn
   356  [pm]
   294  Jake Swanson
    83  Claude
    20  [manager]
     7  [marketer]
```

I’ve been writing code since I first discovered how to copy/paste ActionScript snippets in Macromedia Flash, around 2002 or so. After getting a CS degree in 2012, I dove into a job upgrading Rails 2 applications to Rails 3, and have been doing web development since. But I don’t really have a good handle on what my job title has been the last year. Engineering Support? Delivery Engineer? AI Architect? AI Slop Curator?

A couple months ago our little group (me, a PM, a marketer, and a manager) split off to chase a product idea on its own stack, separate from the mothership. The mandate was simple: stay small, lean on AI hard, and see how far it goes.

## It Goes Fast

We’re sitting around 50k lines of typescript after two months:

```bash
[pm]                 +59171    -14622    net +44549
Jake Swanson         +40563    -9018     net +31545
```

I’m struggling to review and read as much code as I can, while still wanting to ship things (usually complex/risky/opinionated things). And it’s impressive to me that I still get to ship! Thank you Claude for getting the PM’s code right! 🙏

The repo is a server-side rendered monolith you can picture being from that Rails 2 / 3 era, with boring HTML views and as little javascript as we can get away with. As “Engineering Support” I’m trying to reduce the issues coming in. So my goal has been to keep the codebase as simple as possible, because the PM’s AI is merging code to all parts of it:

```bash
# PM's commits
src/views                  +22505    -9483     net +13022
src/services               +12255    -823      net +11432
src/controllers            +10873    -2651     net +8222
src/tasks                  +3884     -118      net +3766
tests                      +2742     -877      net +1865
src/utils                  +1400     -61       net +1339
src/config                 +471      -98       net +373
prisma/migrations          +346      -0        net +346
src/middleware             +327      -93       net +234
prisma/schema.prisma       +296      -42       net +254
```

If you told me a year or two ago that someone who doesn’t know SQL was about to merge ~300 lines of PG columns/tables/etc, I think I would have scoffed at you.

## What Do I Do here?

Most of all, I still think of things daily that I personally want to ship, and I find myself with lots of time and energy to explore those things with an AI assistant that can take on the grunt work. I’m shipping more than I ever have, reading more code than I ever have, and reminiscing over all that code I used to write. Websockets, twilio plumbing, complex PG schema or SQL patterns — there have been some big rocks to build going by. This part of the job feels like I’m prepping a new highway for a big paving machine to come through (the AI). And somehow, as it comes through, I’m sometimes trying to run behind it and drive the roller machine that smooths the road out.

I think of myself as being as much of the engineering department as I can still be, given a galaxy-brained AI developer is in the mix. I read code, and QA product pieces locally as much as I can, while trying to keep things stable. An important part of my day is spent asking the team for new or upcoming priorities. We work in the same room Monday through Friday and have daily standup, and I’m usually trying to stay on top of what we want to build, so I can get any preparatory code committed (sometimes just trying to do the whole thing, I can usually get things close to ready while I’m in there). Rarely I’ll jump in and try to smooth out something Claude one-shotted for the PM. 

## Surprisingly Rare Breakage

Just to reiterate, we’re working in as simple of a setup as I could muster:

- a server-side rendered web app that AI has seen countless times in training data i hope
- a single monolithic git repo with the whole product in it

For the last couple months (with very few customers), pull requests don’t require review to merge and head straight to staging automatically. This would sound absolutely bonkers to 2020 Jake! The closest I’ve gotten is trying to remind people to lean on Claude Opus for complexity checks, since it has the whole product in front of it. So I do review some PRs, and once or twice have requested to take them on and split them up or redo them, but these cases are much rarer than I feared!

There have been rare hiccups. For example, Prisma doesn’t catch schema drift automatically and a couple times Claude managed to ship a new DB column without its migration. I bolted on an automated check for schema drift, which seemed standard in Prisma-land, and it hasn’t happened again.

## Verdict at 2mo

I can keep working this way, and find much of it a breath of fresh air compared to the heavy development processes and organizations I’ve seen in the past. Particularly the velocity! As someone who still gets a lot of joy from delivering software, I am getting those dopamine hits at record rates!
