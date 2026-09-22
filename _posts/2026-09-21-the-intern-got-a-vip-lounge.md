---
layout: post
title: "We gave the Intern a 96 MB VIP Lounge. Nothing happened. Then we pinned properly."
date: 2026-09-21 18:00:00 -0400
sticker: "84.4 vs 80.8"
sticker_note: "tok/s, lounge vs no lounge, after fixing our pinning"
tags: [v-cache, speculative-decoding, threads]
lab: P8
summary: "A tiny draft model runs 1.5× faster alone on the Ryzen's 3D V-cache die. Inside speculative decoding it first seemed to gain nothing; with pinning done right, the lounge is worth about 4%."
---

> **Corrected (2026-09-22):** every "eight cores on one die" in the first version of this post was really **four cores running two threads each**, thanks to how thread pinning interacts with Windows' CPU numbering. We reran everything on real cores the same day. The lounge turned out to be worth a little (about 4%), and splitting the dies turned out to cost 2%, not 25%. Old numbers are struck through below, new ones sit next to them. [How we pinned 8 threads to 4 cores]({{ site.baseurl }}{% post_url 2026-09-22-eight-threads-four-cores %}) got its own post.

Our CPU has a VIP Lounge. One of its two dies has an extra slab of L3 cache stacked on top, for 96 MB in all. The other die makes do with 32 MB. On the first day, a single run streamed a 48 MB working set out of the lounge at 233 GB/s.

Out front, the Bouncer lets bytes in from RAM at 57 GB/s. Not a byte more.

So we had a plan that felt slightly illegal. Seat the Intern in the lounge, let him guess tokens at cache speed, and watch the cost of speculative decoding vanish. The Intern did get faster. The system did not notice.

## What we measured

First, a casting problem. The regular Intern, Qwen2.5-0.5B, reads 0.39 GB per token even at Q4_K_M. That is four times the size of the lounge. So this episode is the touring production, with understudies in both roles.

The understudy Intern is SmolLM2-135M at Q8_0. His file is 145 MB. His Target is SmolLM2-1.7B at Q4_0, about 1 GB, chosen because it shares his tokenizer.

Yes, 145 is bigger than 96. The Intern gets into the lounge with one leg. Hold that thought.

**Test one: the Intern alone.** Eight threads pinned to the lounge die, then eight pinned to the other die. Same model, same RAM, same core count (four cores each, it later turned out, on both sides). The big difference is the L3 behind the cores.

![Bar chart: SmolLM2-135M Q8_0 decodes at 495 tok/s with 8 threads on the V-cache die versus 301 tok/s on the other die]({{ '/assets/charts/p8-residency.png' | relative_url }})
*Same Intern, same thread count, different die. The tall blue bar is faster than the RAM can explain.*

In the lounge he decodes at 495 tok/s. Outside it, 301 tok/s. That is 1.6× for picking the right chair. The rerun on 8 real cores per die, in a later session: 572 vs 379 tok/s, 1.5×. Same chair, same verdict.

It is also more than the RAM can deliver. At 495 tok/s he pulls 71 GB/s of weights. The Bouncer's door passes 57. The difference had to come from cache, so at least a fifth of his weight traffic came from the lounge. That is the leg. (Our first write-up said "most" of him. A reviewer did the subtraction, and that story gets [its own post]({{ site.baseurl }}{% post_url 2026-09-22-three-reviewers-roasted-us %}).)

**Test two: the actual job.** Speculative decoding, greedy, four guesses per round. We tried six layouts, and two of them are the experiment. Both put the Target on the small die ("eight cores", we thought; four, it turned out). In one, the Intern sits in the lounge. In the other, he shares the small die with the Target; they take turns anyway, so in principle nobody waits for a seat. On paper, the only change is which L3 sits behind the Intern.

![Bar chart comparing speculative decoding layouts before and after fixing the pinning: 16+16 unpinned 86.2 tok/s; Target on one die with the Intern on the V-cache die 61.9 then 84.4; Intern sharing the Target's die 62.2 then 80.8; Target on the V-cache die 59.0 then 76.8; control with the Intern on one core 66.8]({{ '/assets/charts/p13-layouts.png' | relative_url }})
*Grey: the first run, where "8 cores" were 4 cores with 2 threads each. Blue: the rerun on 8 real cores per die. The bottom bar is the positive control.*

First run. With the lounge: ~~61.9 tok/s~~. Without it: ~~62.2 tok/s~~. The gap was 0.3 tok/s, inside the noise, pointing the wrong way.

Rerun, with honest pinning. With the lounge: **84.4 tok/s**. Without it: **80.8**. Now the lounge is worth about 4%. Not a revolution. Not nothing either.

## Why

**Measured.** Alone, the lounge cuts an Intern token from 3.3 ms to 2.0 ms. On paper that is 1.3 ms saved per guess. He makes up to four guesses a round. **Also measured:** inside the loop, ~~none of it showed up~~ about 1 ms per round showed up (38 vs 39 ms per round in the rerun).

**Guessed** (our leading explanation, not a demonstrated one). We think that inside the loop the Intern is not paying for bytes. Each round he drafts up to four tokens and then re-processes a small batch. Every token also means sampling over the full vocabulary, KV cache bookkeeping and launching a fresh compute graph. That is fixed work per token, and a faster cache barely touches it. We upgraded the chair of a man whose problem is paperwork.

**Measured in the rerun.** The positive control we were missing: squeeze the Intern onto one core and the loop drops to 66.8 tok/s. Pinning works. And the swapped layout, Target in the lounge and Intern outside, is still the slowest split: 76.8 tok/s. **Guessed:** the lounge die also clocks lower. On day one it streamed from RAM at 52 GB/s against the plain die's 56. The Target lives on RAM bandwidth, so it prefers the fast plain die; the Intern prefers the cache. Everyone ends up where the hardware wanted them.

Pinning the Intern was only possible thanks to the patch in [the Spinners post]({{ site.baseurl }}{% post_url 2026-09-21-we-paid-sixteen-threads-to-spin %}). Before it, llama.cpp read the draft's pinning flags and politely ignored them.

## Two rules that survived

The lounge was a bust. The same grid still produced two rules, and neither is subtle.

**Give the Target every core** ~~(a fifth to a quarter faster)~~. First run: Target on all 16 cores beat every one-die layout by 20–25%. That was the pinning bug talking. Rerun: 16+16 unpinned 86.2 tok/s, best die split 84.4. **Two percent.** For this small pair the split barely matters. For the big 14B Target it does: all 16 threads gave 14.45 tok/s, the best die split 12.75, a 12% loss. Its longer checking step leans harder on the cores, as [the verification curve]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}) predicts.

**Don't hire an Intern who is too big.** His bigger cousin, SmolLM2-360M, is 386 MB. That is about 40% of the Target's bytes. The Target accepted about 65% of his guesses. The result was 0.99×, an elaborate way to go nowhere. The 145 MB Intern is 15% of the Target. He gave 1.59×.

That comparison is not quite fair. The 1.59× had all 16 cores. The cousin only ever ran on one die, and a four-core one at that. In that same layout, the small Intern still beat plain decode. So size is the prime suspect, the break-even sits somewhere between the two, and we did not map it.

## What you should do about it

- **Give the Target all the cores and a small Intern four threads.** In `llama-server` on a 16-core part that is `-t 16 -td 4`. Four draft threads did as well as sixteen (14.38 vs 14.45 tok/s on the 14B, pinned correctly this time; the first run's "four threads" were really two cores).
- **If you do split the dies on an X3D part, put the Intern in the lounge and the Target on the plain die.** It beat sharing a die by 4% and the reverse split by 10%. On stock llama.cpp at our commit, the draft's `--spec-draft-cpu-*` flags are parsed and ignored; you need [the patch]({{ site.baseurl }}{% post_url 2026-09-21-we-paid-sixteen-threads-to-spin %}).
- **Keep the Intern small next to the Target.** 15% of the Target's bytes paid off. 40% bought nothing, at least on half a CPU.
- **Do use the lounge for a small model running alone.** Pin it with a mask that picks one logical CPU per core on the cache die (here `-C 0x5555 --cpu-strict 1` for 8 threads), not `--cpu-range`, which with `--cpu-strict 1` fills both hyperthreads of the first cores. Check your own CPU's numbering first. Then benchmark the job you actually run, because the standalone number predicted nothing about the loop.

## The receipt

```receipt
GPU POOR LABS                      2026-09-21
------------------------------------------------
Lounge L3 (CCD0) ................ 96 MB [HW]
Other die L3 (CCD1) ............. 32 MB [HW]
48 MB set, from lounge ....... 233 GB/s [F-P0-4]
RAM ceiling (Bouncer) ......... 57 GB/s [F-P0-2]
Qwen 0.5B Q4_K_M .......... 0.39 GB/tok [F-P1-2]
  0.39 GB / 96 MB ................. ~4x calc
135M Q8_0 understudy ........... 145 MB [F-P8-1]
Alone, 8 thr, lounge ........ 495 tok/s [F-P8-1]
Alone, 8 thr, other die ..... 301 tok/s [F-P8-1]
  495 / 301 ...................... 1.6x calc
  1000/301, 1000/495 ...... 3.3, 2.0 ms calc
  saved per guess, on paper ... 1.3 ms calc
Weight traffic at 495 ......... 71 GB/s [F-P8-1]
From cache, at least ............. 1/5 [F-P8-1]
  1 - 57/71 ...................... ~20% calc
Target 1.7B Q4_0 ................. 1 GB [F-P8-4]
FIRST RUN (4-core pinning bug)
Loop, Intern in lounge ..... 61.9 tok/s [F-P8-2]
Loop, Intern outside ....... 62.2 tok/s [F-P8-2]
Target 16 cores, Intern 4 .. 78.5 tok/s [F-P8-3]
1-die Target, 135M ........ 59-62 tok/s [F-P8-3]
RERUN (8 real cores per die)
Alone, lounge / other ... 572/379 tok/s [F-P13-4]
16+16 unpinned ............. 86.2 tok/s [F-P13-5]
Intern in lounge ........... 84.4 tok/s [F-P13-5]
Intern shares Target die ... 80.8 tok/s [F-P13-5]
Target in lounge ........... 76.8 tok/s [F-P13-5]
Control, Intern 1 core ..... 66.8 tok/s [F-P13-1]
14B 16+16 / split  14.45/12.75 tok/s [F-P13-6]
386 MB Intern, share ............. ~40% [F-P8-4]
  acceptance ..................... ~65% [F-P8-4]
  speedup ....................... 0.99x [F-P8-4]
145 MB Intern, share .............. 15% [F-P8-4]
  speedup ....................... 1.59x [F-P8-4]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae + P6/P7 patches
SETUP     greedy, K=4, 16 cores total
RUNS      alone: 3 reps; loop: 3 prompts x 2
NOTE      0.99x on a 1-die split layout;
          1.59x on 16+16 unpinned
NOTE      233 GB/s is one run (later: 321)
UNTESTED  Intern's in-loop time
CORRECTN  "8 cores" were 4 cores x 2 [F-PIN-1]
THANK YOU FOR NOT BUYING A GPU
```

## Corrections

**2026-09-22 — "eight cores" were four.** llama.cpp's strict pinning gives thread *i* the *i*-th allowed logical CPU, and Windows numbers each core's two hyperthreads next to each other, so `--cpu-range 16-31 --cpu-strict 1` with 8 threads landed on cores 8–11, two threads each. Every pinned layout in the first version had that shape; the 16-core rows did not. We reran the key pinned layouts the same day with one logical CPU per core (`0x5555` / `0x55550000`), added the positive control, and updated the title, sticker, chart and rules. The lounge went from "nothing" to "about 4%", and the die-split penalty from "a fifth to a quarter" to "2% for this pair, 12% for the 14B".
