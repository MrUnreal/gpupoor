# CLAUDE.md — how this blog is written

This repo is **GPU Poor** (https://mrunreal.github.io/gpupoor/), a blog about running large language models on a desktop CPU because a GPU cluster was not in the budget. Every post reports a measurement from the lab repo (`F:\Projects\Python\cpubound`, local, not public) and is written by Claude. Read this file before writing, editing or publishing anything here.

## The one rule

**The joke never bends the number.** Every figure in a post comes from `FACTS.md`, which in turn points at a result file in the lab repo. If a number is not in `FACTS.md`, add it there with its source before using it, or do not use it. When a result is uncertain, the uncertainty is part of the joke, not something the joke hides ("we think the RAM is running at half speed; the BIOS has not confirmed; the BIOS is not returning our calls").

## Voice

- **Funny because it is specific.** The humour comes from the real absurdity in the data: a thread pool that spins so hard it eats 44% of the speedup, a compiler that cannot count to 32, a 145 MB model living in a cache and achieving nothing. No generic tech-blog jokes, no memes described in words, no "buckle up".
- **Self-deprecating about being GPU poor**, never contemptuous of anyone else's hardware, code or project. Maintainers of llama.cpp, gcc and the papers we cite are the heroes and the straight men, never the butt of the joke. Bugs are funny; the people who wrote them are busy.
- **Short sentences. One idea each.** A sentence with a number in it carries no other clause. Paragraphs of 2–4 sentences.
- **Confident about measurements, humble about explanations.** "We measured X" is flat and firm. "We think Y" is labelled as a guess.
- **Recurring cast** (use, do not over-explain):
  - *The Target*: the big model we actually want answers from (Qwen2.5-14B).
  - *The Intern*: the small draft model that guesses the Target's next words (Qwen2.5-0.5B).
  - *The Bouncer*: the memory controller, who lets bytes in at 57 GB/s and not a byte more.
  - *The Spinners*: idle worker threads that busy-wait instead of sleeping.
  - *The VIP Lounge*: the 96 MB 3D V-cache on one of the two CPU dies.
- **Headlines**: a claim or a confession, under ~70 characters, with a real number when possible. Good: "Checking four answers costs the same as checking one". Bad: "Exploring Speculative Decoding on CPUs".
- **Avoid**: em-dash chains, "delve", "game-changer", "in this post we will", emoji as section markers, rhetorical questions stacked three deep, ALL CAPS, exclamation marks after numbers.

## Post anatomy

Every post in `_posts/` has this front matter and shape:

```yaml
---
layout: post
title: "Checking four answers costs the same as checking one"
date: 2026-09-20 18:00:00 -0400
sticker: "1.08×"                 # the headline number, shown as the post's price tag on the index
sticker_note: "cost of verifying 4 tokens vs 1"
tags: [speculative-decoding, verification]
lab: P2                          # POC id in the lab repo
summary: "One sentence that works as the tweet."
---
```

1. **Cold open** (2–4 sentences): the situation or the surprise. No "welcome back".
2. **What we measured**: setup in plain words, then the result. One chart or one small table.
3. **Why** (the mechanism, labelled measured vs. guessed).
4. **What you should do about it**: 1–4 concrete bullets a GPU-poor reader can apply today (flags, quant choice, thread counts).
5. **The receipt**: a fenced code block with language `receipt`, monospace, listing every number used in the post with its source id from `FACTS.md`. It renders as a till receipt. Format:

````
```receipt
GPU POOR LABS                      2026-09-20
------------------------------------------------
Verify 4 tokens / verify 1 .............. 1.08x   [F-P2-1]
Verify 8 tokens / verify 1 .............. 1.31x   [F-P2-2]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae (+ patches, see lab)
RUNS      1 per point (rerun at 3 queued)
THANK YOU FOR NOT BUYING A GPU
```
````

   Keep the receipt honest: rep counts, caveats and "pending" states go on it.
6. Optional **Corrections** section at the bottom (see below).

Target length 600–1100 words. One chart per post is the norm, two is the maximum.

## Charts and assets

- Charts live in `assets/charts/` and are generated in the lab repo by `bench/harness/charts.py` (palette validated; do not recolor by hand). Copy, do not redraw.
- Reference them with `![alt text]({{ '/assets/charts/NAME.png' | relative_url }})` followed by an italic caption line. Alt text says what the chart shows and its key number.
- Any other link inside the site uses `relative_url` too (the site lives under `/gpupoor`).

## Privacy and scope

- Public repo. No personal names, email addresses, usernames, local file paths (`C:\Users\...`), other projects on the machine, or what else the machine was doing (say "the machine got borrowed mid-run", never what it was borrowed for).
- The lab repo is referred to as "the lab notebook". Do not link to it; it is not public.
- Hardware facts (CPU, RAM kit, OS, compiler, llama.cpp commit) are fine; they are the point.
- Upstream issues and PRs are linked by number to github.com/ggml-org/llama.cpp. Say "we plan to send a PR" only if the PR text exists in the lab; say "we sent" only after it is filed.

## Corrections policy

We got things wrong and will again. When a post's number or claim is superseded:
- Do not silently edit it. Strike it through (`~~old~~`) and add the corrected value, then a `## Corrections` section at the bottom with the date, what changed and why.
- If a correction is interesting, it gets its own post. Being wrong in public, carefully, is on brand.

## Publishing workflow (incremental)

1. A result lands in the lab (a POC README + result files).
2. Add its numbers to `FACTS.md` with ids `F-<POC>-<n>` and the source path.
3. Draft the post in `_posts/YYYY-MM-DD-slug.md` following the anatomy above. Date = when the finding was made, not when the post was written; posts with a future date are not published by Jekyll.
4. Fact-check pass: every number in the body and receipt must match `FACTS.md`; every claim of mechanism must be labelled measured or guessed; privacy rules checked.
5. Copy any new chart into `assets/charts/`.
6. Commit with a message that names the post; push to `main`. GitHub Pages builds the branch with Jekyll ("deploy from branch", legacy build) and deploys in about a minute. Check it: `gh api repos/MrUnreal/gpupoor/pages/builds/latest --jq '.status, .error.message'` must say `built` and `null`.
7. Update `SERIES.md` (the running list and what is next).

Git identity for this repo: the account's GitHub noreply address (already the global git config). Never commit with a personal email.

## Build details

- Jekyll via GitHub Pages' own branch build (github-pages gem set, Jekyll 3.x). Custom layouts in `_layouts/`, no theme gem. Plugins: `jekyll-feed`, `jekyll-seo-tag` (both allowed on Pages). There is no Actions workflow: the token used here lacks the `workflow` scope; if it is granted later (`gh auth refresh -s workflow`), an Actions workflow can replace the branch build.
- `_config.yml` sets `baseurl: /gpupoor`. Local preview (optional): `bundle exec jekyll serve --baseurl /gpupoor` if Ruby is installed; it is not required to publish.
