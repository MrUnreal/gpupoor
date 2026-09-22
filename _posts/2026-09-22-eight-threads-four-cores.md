---
layout: post
title: "We hired eight movers. It was four guys in two hats each."
date: 2026-09-22 15:20:00 -0400
sticker: "8 = 4"
sticker_note: "threads we pinned vs cores we got"
tags: [threads, pinning, corrections]
lab: P13
summary: "For two days every 'eight cores on one die' result on this blog ran on four cores with two threads each. A fact-checker caught it. The rerun overturned one finding, softened another and left the rest standing."
---

For two days we ran experiments on "one die, eight cores". We split the Target and the Intern across the two dies of our Ryzen. We measured how much the split cost. We wrote it up. It cost a fifth to a quarter of the speed, we said, so give the Target every core.

Then a fact-checker, auditing the one-die curve in our verification-cost post, read llama.cpp's thread code. The question that came back was rude. How many physical cores did those eight threads actually get?

Four.

## How eight became four

Our CPU has 16 cores and 32 logical CPUs, because each core runs two hardware threads. Windows numbers them in pairs: core 0 is CPUs 0 and 1, core 8 is CPUs 16 and 17. So far, so normal.

To pin llama.cpp to the second die we wrote `--cpu-range 16-31 --cpu-strict 1` with 8 threads. That range is all 16 logical CPUs of die two. We assumed eight threads would spread across it. Strict mode does not spread. It hands each thread the next allowed CPU in order: 16, 17, 18 and so on up to 23. CPUs 16 to 23 are cores 8, 9, 10 and 11, two threads each.

We hired eight movers. We got four guys, each wearing two hats.

**Read, not guessed:** the placement is in llama.cpp's source, and Windows' own processor map confirms the pairs. The most embarrassing detail: our own memory benchmark from day one pinned every other CPU, correctly. We knew the numbering. We just did not tell llama.cpp.

The fix is a mask with one bit per core: `-C 0x55550000 --cpu-strict 1` for the eight real cores of die two, `0x5555` for die one. We reran the key pinned layouts the same day.

## What the rerun changed

![Bar chart comparing speculative decoding layouts before and after fixing the pinning: 16+16 unpinned 86.2 tok/s; Target on one die with the Intern on the V-cache die 61.9 then 84.4; Intern sharing the Target's die 62.2 then 80.8; Target on the V-cache die 59.0 then 76.8; control with the Intern on one core 66.8]({{ '/assets/charts/p13-layouts.png' | relative_url }})
*SmolLM2-1.7B Target, SmolLM2-135M Intern, four guesses per round. Grey: the first run, four guys in two hats per die. Blue: the rerun two days later, eight real cores per die. The runs are different sessions: all 16 threads unpinned did 78.2 in the first one. Compare bars within a colour, not across.*

**Overturned: "splitting the dies costs a fifth to a quarter."** First run: every one-die layout lost 20–25% against all 16 threads. Rerun: all 16 threads unpinned did 86.2 tok/s. The best split did 84.4. That is 2%.

**Softened: "the VIP Lounge does nothing in the loop."** First run: 61.9 tok/s with the Intern in the lounge, 62.2 with him sharing the Target's die. Rerun: 84.4 and 80.8. About 4%. [That post]({{ site.baseurl }}{% post_url 2026-09-21-the-intern-got-a-vip-lounge %}) now carries both sets of numbers, the old ones struck through.

**Survived, awkwardly: the one-die checking cost.** In the two-hat run, checking 8 guessed tokens cost 1.60× a one-token round. That was a single run. The rerun on eight real cores, three reps, gave 1.65–1.88×. Twice the real cores did not make checking any cheaper relative to one token. **Guessed:** in these kernels, a core's second hardware thread hides about as much waiting as a second core adds arithmetic. Or something else on the machine got in the way. We did not separate the two. [That post]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}) has the new curve.

**Survived: the Intern alone in the lounge.** 572 tok/s on the cache die against 379 on the other, with eight real cores each. That is 1.5×. The first run, in an earlier session, said 1.6×. Both sides then had the same four guys in hats, so that comparison was fair all along.

**Survived intact: almost everything else.** Nearly every headline number on this blog ran on 16 unpinned threads. The pinning bug never touched them.

## The control we should have run first

Our reviewers had asked for it earlier the same day: prove the pinning works by pinning something somewhere it must hurt. So we squeezed the Intern onto a single core. The loop fell from 84.4 tok/s to 66.8. Pinning works. It just has to be told the truth.

## The big Target is different

For the 14B Target and its 0.5B Intern, the split still hurts. All 16 threads: 14.45 tok/s. Target on one die, Intern on the other: 12.75. That is 12% slower. **Guessed:** a 14B round spends far longer checking, and checking is where extra cores earn their keep.

A 4-thread Intern on four real lounge cores, next to a 16-thread Target: 14.38 tok/s. The Intern still needs only four threads.

## What you should do about it

- **Before you pin anything, look up your CPU numbering.** On this Windows machine each core's two threads are neighbours. Other systems can number them differently. On Linux, `lscpu -e` shows which core each logical CPU belongs to.
- **Pin with a mask that has one bit per core** if you want one thread per core. On this machine, `--cpu-range` plus `--cpu-strict 1` packs threads onto both hardware threads of each core.
- **Never pin 16 threads with a full strict mask** on a 16-core part numbered like ours. It packs all 16 onto the first 8 cores. Leave a 16-thread run unpinned.
- **Run a positive control.** Pin something where it must be slow and check that it is.

## The receipt

```receipt
GPU POOR LABS                         2026-09-22
------------------------------------------------
Threads pinned per die ............. 8 [F-PIN-1]
Physical cores they got ............ 4 [F-PIN-1]
Mask die 1 (lounge) ........... 0x5555 [F-PIN-2]
Mask die 2 ................ 0x55550000 [F-PIN-2]
FIRST RUN, 4 cores x 2 threads [F-PIN-1]
16+16 unpinned ............ 78.2 tok/s [F-P8-3]
One-die layouts .......... 59-62 tok/s [F-P8-3]
Split vs all-16 penalty ....... 20-25% [F-P13-7]
Loop, lounge / sharing ..... 61.9/62.2 [F-P8-2]
Intern alone, lounge/other ... 495/301 [F-P8-1]
1 die, verify 8, 1 run ......... 1.60x [F-P2-3]
RERUN, 8 real cores per die [F-PIN-2]
16+16 unpinned ............ 86.2 tok/s [F-P13-5]
Best split ................ 84.4 tok/s [F-P13-5]
Intern on Target's die .... 80.8 tok/s [F-P13-5]
Target in lounge .......... 76.8 tok/s [F-P13-5]
Split cost / lounge gain ..... 2% / 4% [F-P13-5]
1 die, verify 8, 3 reps ... 1.65-1.88x [F-P13-3]
Intern alone, lounge/other ... 572/379 [F-P13-4]
Intern alone, ratio ............ 1.51x [F-P13-4]
Control: Intern, 1 core ... 66.8 tok/s [F-P13-1]
14B all-16 / split ....... 14.45/12.75 [F-P13-6]
14B split cost ................... 12% [F-P13-6]
14B Target 16, Intern 4 .. 14.38 tok/s [F-P13-6]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae + P6/P7 patches
RUNS      loops 3 prompts x 2 reps; Intern
          alone and verify curve 3 reps
SESSIONS  first run 2026-09-20, rerun 09-22
CAUGHT BY a fact-checker, not by us
THANK YOU FOR NOT BUYING A GPU
```
