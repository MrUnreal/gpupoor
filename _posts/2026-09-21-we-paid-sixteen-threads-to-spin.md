---
layout: post
title: "We paid 16 threads to spin in place. They ate 44% of our gain."
date: 2026-09-21 10:00:00 -0400
sticker: "1.58× → 2.03×"
sticker_note: "same models, fewer Spinners"
tags: [speculative-decoding, threads, llama-cpp]
lab: P6
summary: "llama.cpp's idle worker threads jog in place on the cores the draft model needs. Two flags, or a small patch, took our CPU speculative decoding from 1.58× to 2.03×."
---

Speculative decoding on this CPU should be worth about 2×. With llama.cpp's default flags, it gave us 1.58×. Same Target, same Intern, same prompts, same guesses accepted. Something between the two models was eating time, and it left footprints on every single round.

If you read [the Intern's job interview]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}), its 2× was measured with the culprits already sedated. This is the story of how they got sedated.

## What we measured

The Target (Qwen2.5-14B, Q4_K_M) and the Intern (Qwen2.5-0.5B, Q8_0) share all 16 cores inside `llama-server`. The Intern drafts four tokens, the Target checks them in one pass, repeat. Greedy decoding, three prompts (code, prose, a list), speedup measured against the Target writing alone.

With default flags, each model gets 16 threads. The speedup was 1.58×.

Then we added two flags: `--poll 0 --spec-draft-poll 0`. The speedup was 2.03×.

One round (the Intern drafts, the Target checks) took about 300 ms before. It took 231 ms after. The Intern's acceptance did not move at all. Only the clock changed.

Put it as a budget. With idle threads told to sleep, the pair ran at 2.03× the Target's own speed. The Intern's contribution is the part above 1×, so 1.03. The default setup kept 0.58 of it. The missing 0.45 is 44% of the gain.

![Bar chart of speedup over plain decode: stock defaults 1.58×, stock with the two poll-0 flags 2.03×, patched defaults 2.03×, and two split-die layouts (pinned to 4-core Targets) well below 2×]({{ '/assets/charts/p6-thread-layout.png' | relative_url }})
*Mean of three prompts, K = 4. The flags and the patch land on the same number. The two split-die bars are not in our ledger, so they stay off the receipt; they also turned out to pin the Target to [four cores, not eight]({{ site.baseurl }}{% post_url 2026-09-21-the-intern-got-a-vip-lounge %}#corrections), and are being remeasured.*

## Why: a whodunit

**Suspect one: the Intern.** Alibi: its acceptance was identical in every line-up (measured). It guessed exactly as well each time. It may have been kept waiting for a chair, but waiting is not a crime.

**Suspect two: the Bouncer.** Alibi: same models and same accepted tokens mean the same bytes through the door each round (inferred; we did not count bytes separately). The memory controller was asked to do nothing new.

**Suspect three: the Spinners.** No alibi. Visibly out of breath.

Here is what the code says (read, not guessed). Without OpenMP, llama.cpp gives the Target a persistent pool of worker threads. When a job finishes, the idle workers do not go to sleep right away. They busy-wait first, polling for the next job, for a spin budget set by `--poll`. The bet is that the next job arrives soon and that waking a sleeper costs time. When one model owns the machine, that is a sensible bet.

Speculative decoding is two models taking turns on the same cores. The Target finishes checking. Its 16 workers start jogging in place. Then the Intern shows up to draft on those very cores.

**Twist one: the Intern had no desk.** Its context was created without a thread pool. So every draft token took ggml's disposable path: create OS threads, run one graph, join them, goodbye. The Intern hired a fresh crew of temps for every token it wrote.

**Twist two: the Intern's requests went straight into a drawer.** `--spec-draft-poll`, `--spec-draft-cpu-range` and `--spec-draft-prio` were parsed, then ignored. Only the thread count made it across. A control run agrees: `--spec-draft-poll 0` on its own measured the same as the defaults. A sleep setting for a pool that does not exist is a note taped to a door that is not there.

So, on paper, fresh temps and pacing Spinners want the same 16 cores at the same time. `--poll 0` sends the Target's workers straight to sleep between jobs. The time came back (measured).

Our guess, not profiled: most of the missing ~70 ms per round was the Intern's temps queueing behind the Spinners for a core. The hiring and firing itself looks cheap. The flag fix still hires a fresh crew for every token. It reaches 2.03× anyway.

## The fix

Flags that only work if you already know about them are a trap for the next person. So we wrote a small patch:

- The Intern gets its own persistent pool, built from its own CPU flags, so those flags finally reach something. (Pass `-td` as well; without it, the parser still copies the Target's settings over the Intern's.)
- Before either model runs, the other model's pool is parked.

The pause button already existed. ggml ships `ggml_threadpool_pause`, and a paused pool wakes itself up on its next job. The maintainers built the button; we only wired it to the door.

With the patch and default flags, the speedup was 2.03×. Adding the poll flags on top changed nothing. A patch that removes exactly the contention, and lands on exactly the flag-tuned number, is strong circumstantial evidence. It is not a confession.

Now the caveats. The stock and patched runs were not interleaved, and a rerun that alternates them is queued. The patched speedup divides by the stock session's plain-decode baseline; the patched binary's own came in lower, so this flatters the patch less, not more. The patch targets `GGML_OPENMP=OFF` builds; in OpenMP builds the pause does nothing, and we did not measure one. The PR text is written and not yet filed; it waits for that rerun. An open upstream issue is about letting the two models share one pool ([#27039](https://github.com/ggml-org/llama.cpp/issues/27039)). If that lands, our patch should happily dissolve into it.

## What you should do about it

- **CPU speculative decoding with a draft model, non-OpenMP build:** add `--poll 0 --spec-draft-poll 0`. That is our whole jump from 1.58× to 2.03×. On a stock build the second flag is decorative, but it is what we measured.
- **Do not trust the `--spec-draft-*` CPU flags on a stock build.** Pinning, priority and polling for the draft are parsed and ignored. If you pinned your draft somewhere clever and saw no effect, you measured nothing. We did exactly that. Our later runs on the patched build, where the flags finally reach a pool, are in [the VIP Lounge post]({{ site.baseurl }}{% post_url 2026-09-21-the-intern-got-a-vip-lounge %}).
- **Know which thread pool you built.** This story is about ggml's own pool. On Windows with gcc you want that one anyway, [because the OpenMP build is the slow one there]({{ site.baseurl }}{% post_url 2026-09-20-openmp-tax %}).
- **Leave `--poll` alone for plain decode.** Our one control hinted that `--poll 0` might cost plain decode a little. It landed inside the day's noise, so we have no verdict. A proper control is queued with the rerun.

## The receipt

```receipt
GPU POOR LABS                      2026-09-21
------------------------------------------------
Stock, default flags ........ 1.58x  [F-P6-1]
Stock + both poll-0 flags ... 2.03x  [F-P6-1]
ms/round, spin -> sleep ~300->231    [F-P6-1]
Patched, default flags ...... 2.03x  [F-P6-2]
Draft pool before patch ...... none  [F-P6-3]
Draft CPU flags before ..... ignored [F-P6-3]
Intern post (pools asleep) 2.0-2.2x  [F-P3-1]
------------------------------------------------
Gain on offer 2.03-1 .................. 1.03x
Gain delivered 1.58-1 ................. 0.58x
Gain missing 1.03-0.58 ................ 0.45x
Gain eaten 0.45 / 1.03 .................. 44%
Round time saved ~300-231 ............ ~70 ms
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae, gcc 14,
          GGML_OPENMP=OFF (+ P6 patch)
MODELS    14B Q4_K_M + 0.5B Q8_0, K=4, greedy
RUNS      1 per prompt, 3 prompts per row
          stock vs patched not interleaved
          (interleaved rerun queued)
BASELINE  stock-session plain decode, all rows
PR        text written, not filed
THANK YOU FOR NOT BUYING A GPU
```
