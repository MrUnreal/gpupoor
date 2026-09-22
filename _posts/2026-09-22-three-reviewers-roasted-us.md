---
layout: post
title: "We asked three AIs to roast our lab notes. They found 3 blockers."
date: 2026-09-22 13:00:00 -0400
sticker: "3 blockers"
sticker_note: "and a pile of majors"
tags: [corrections, methodology, peer-review]
lab: ADR-0011
summary: "Before this blog went up, three AI reviewers read the whole lab notebook. They found a peak from the wrong CPU generation, a quality score that partly measured itself, and a PR title describing the fix that failed."
---

Before this blog went up, we handed the whole lab notebook to three AI reviewers and asked them to be unkind. One reads methods sections. One reads microarchitecture manuals. One reads pull requests the way a maintainer with a long queue does.

They came back with three blockers, a pile of majors, and no emotional attachment to any of our numbers. The figures in the earlier posts are the corrected ones. This is the post about how they got corrected.

## What we measured

Nothing ran for this post. The experiment was us. Each reviewer read every write-up and the result files behind it, and tagged each problem blocker, major or minor. A blocker means the sentence does not ship.

Here is the damage, lightly edited for dignity.

| We wrote | What is actually true | Tag |
|---|---|---|
| Prefill runs "near the AVX-512 peak" | About 40% of it | blocker |
| Speculative output is as good as greedy, "measured properly" | The scorer re-tokenized text and used one kernel path | blocker |
| PR title: pass the table "by reference" | The patch passes a pointer | blocker |
| The MoE's missing 28% is overhead | Best guess now: bytes we never counted | major |
| "Most" of a small draft is served from cache | At least a fifth | major |
| The 57 GB/s wall is "the memory controller" | Unproven when written | major |
| Speculative decoding gives 2.02× | 2.0–2.2×, depending on the day | major |

**Blocker one: the peak.** At Q4_K_M, our 7B model prefills at 155 tok/s. That is about 2.2 TFLOP/s-equivalent. We held it up against the AVX-512 FP32 peak and wrote "near". The peak we used belonged to the previous core generation. The real one for these 16 Zen 5 cores is about 5 TFLOP/s. We were at about 40%.

"Near" was doing a lot of work. The good news hid inside the bad news, though: 40% means headroom. An open llama.cpp PR, [#27851](https://github.com/ggml-org/llama.cpp/pull/27851), goes after it. With that PR and `--no-repack`, [7B Q4_K_M prefill ran about 2.5× faster than stock]({{ site.baseurl }}{% post_url 2026-09-22-q4-0-vs-q4-k-m-rematch %}). That setting has two runs so far.

**Blocker two: the quality score.** Speculative decoding should give the Target's own answers. We checked by scoring every output under the Target. The scorer took the output *text* and tokenized it again. Text does not always come back as the tokens that produced it. Every mismatch counted as the model disagreeing with itself.

It also scored everything through the batched kernel. That is the same path speculative verification runs on. The ruler was cut from the same wood as one of the things it measured.

The v1 numbers exist. Log-probability came out at −0.21 to −0.24 nats per token. Argmax agreement was 95–98%. We had called the speculative outputs "equal or slightly better". What those numbers actually measure is now an open question. Version 2 scores the stored token ids through both kernel paths, and it is pending.

**Blocker three: the PR title.** Our fix for [the Q4_0 crash]({{ site.baseurl }}{% post_url 2026-09-21-gcc-cannot-count-to-32 %}) has a PR text, not yet filed. Its title said we pass a lookup table by reference. The body of the same PR explains that passing it by reference does not fix the crash. The patch passes a pointer. A maintainer reads the title first. Ours advertised the fix that failed. It now says pointer.

**The majors, speed round.**

- We blamed the MoE's missing 28% on overhead. A reviewer read the model file's header instead. 30 of its 41 layers each carry a 2 MiB recurrent state. That state is read and written every token, and our bytes-per-token math never counted it. Leading hypothesis only; the per-op profile has not run.
- A 145 MB draft model, the Intern's understudy, ran alone in the VIP Lounge. Its effective read rate was 71 GB/s. DRAM stops at 57 GB/s. We wrote that "most" of it came from cache. The arithmetic only proves at least a fifth.
- We cast the memory controller as the Bouncer on a hunch. The casting came before the evidence.
- A draft of K tokens gets verified as K + 1. In places we priced K. The off-by-one: a classic, in a new venue.
- Result files from patched builds recorded only the upstream commit. The records could not tell stock from patched. Neither could anyone reading them.

## Why

**Measured.** Plain decode of the same 14B model on the same machine ranged 6.24–6.70 tok/s across one day. That is about 7% of weather. The same speculative setup measured 2.02× in one session. It measured 2.19× in another. Several of our speedups divided a speculative run by a plain baseline from a different session. A difference smaller than the weather is a coin toss, and we had been reporting coin tosses to two decimal places.

**Guessed.** We think most of the errors share one shape. The numbers were mostly fine. The adjectives around them got promoted. "Consistent with" became "confirmed". "At least a fifth" became "most". "A suspect" became "the memory controller". Nobody lied. A good number just makes the sentence next to it feel braver than it is.

In fairness to ourselves, briefly: the reviewers also checked the bandwidth kernel, the patches and the residency arithmetic, and found them sound. That part is going on the fridge.

## What you should do about it

Here is what we changed, translated for your machine.

- **Measure your own weather.** Run the same plain decode in the morning and again at night. We no longer interpret differences under ~5% between sessions. Your machine may need a wider margin.
- **Bracket every A/B.** Baseline, then your configuration, then baseline again, in one session on one build. Divide by the mean of the bracket. Quote the result as a range.
- **Look up the peak for your exact core** before writing "near peak". Then ask whether FP32 is even the right unit; the quantized prefill kernels here are int8 dot products.
- **Score token ids, not text.** If you check speculative quality, keep the ids the model emitted and score them through more than one kernel path. And when a number is wrong in public, strike it through and date the fix. Never edit it quietly.

One suspect is still at large. We have since collected some evidence against the Bouncer. If the memory controller really runs at half clock, every absolute decode number on this blog is about 1.3× low. That is unconfirmed. The ratios should survive either way. [The next post]({{ site.baseurl }}{% post_url 2026-09-22-is-our-ram-at-half-speed %}) lays out the case. Only the BIOS can rule on it.

## The receipt

```receipt
GPU POOR LABS                      2026-09-22
------------------------------------------------
Reviewers / blockers ........... 3 / 3 [REVIEW]
Prefill 7B Q4_K_M .......... 155 tok/s [F-P1-6]
  TFLOP/s-equivalent ............. 2.2 [F-P1-6]
  FP32 peak, 16 x Zen 5 ...... ~5 TF/s [F-P1-6]
  share of peak ................. ~40% [F-P1-6]
7B Q4_K_M prefill, #27851
  + no-repack vs stock ......... ~2.5x [F-P11-1]
  (no-repack: 2 runs)
Quality v1, nats/tok .... -0.21..-0.24 [F-Q-2]
Quality v1, argmax agree ...... 95-98% [F-Q-2]
  (method flawed; v2 pending)
MoE gap vs dense 3B .............. 28% [F-P5-1]
Layers with 2 MiB state ........ 30/41 [F-P5-3]
  (as the cause: hypothesis)
145 MB draft alone, V-cache .. 71 GB/s [F-P8-1]
DRAM read ceiling ............ 57 GB/s [F-P0-2]
From cache >= 1 - 57/71 ......... 0.20 [derived]
Plain 14B, one day, tok/s .. 6.24-6.70 [ledger]
  6.70 / 6.24 .................. 1.07x [derived]
Same cell, two sessions ... 2.02/2.19x [F-P3-1]
  quoted as ................. 2.0-2.2x [F-P3-1]
Noise floor between sessions ..... ~5% [ledger]
Decode low if RAM ctrl at half . ~1.3x [F-P10-4]
  (unconfirmed; BIOS not checked)
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae (+ patches, see lab)
RUNS      none new; reruns queued: P2 x3,
          P6 interleaved, quality v2
PENDING   MoE per-op profile, BIOS clock check
THANK YOU FOR NOT BUYING A GPU
```
