---
layout: post
title: "Q4_0 hit 3.0× on code. Then Q4_K_M found a flag nobody would guess"
date: 2026-09-22 14:30:00 -0400
sticker: "3.0×"
sticker_note: "best run of the study"
tags: [quantization, speculative-decoding, prefill]
lab: "P9, P11"
summary: "On our CPU, Q4_0 is the faster 4-bit model for writing (best run: 3.0× on code with a 0.5B draft); Q4_K_M wins long prompts, but only with an open llama.cpp PR and --no-repack."
---

The usual advice for a 4-bit model is short: get Q4_K_M, it is the sensible one. Q4_0 is the older, plainer format, and on our Windows build it used to [segfault on contact]({{ site.baseurl }}{% post_url 2026-09-21-gcc-cannot-count-to-32 %}). Once it stopped crashing, it had earned a fair fight. We held a rematch in two rounds. Q4_0 won the first on points. Q4_K_M won the second by knockout, wearing a borrowed glove.

## What we measured

**Round 1: writing.** Both fighters are the Target, Qwen2.5-14B, quantized each way. The Intern (Qwen2.5-0.5B) drafted for both. Three prompts (code, prose, a structured list), both Targets back to back in one session, so the RAM's mood of the day had little time to change. Our first attempt went in the bin because the machine got borrowed mid-run.

With the Intern drafting four tokens per step, the Q4_0 Target averaged 15.7 tok/s.
The Q4_K_M Target averaged 14.5 tok/s.
That is 8% more text from the same Intern.
The speedup over each Target's own plain decode was a draw.
Q4_0 got 2.22×.
Q4_K_M got 2.19×.

Then we let the Intern draft eight tokens at a time on the code prompt. Plain Q4_0 decode was 7.0 tok/s. With the Intern it was 21.3 tok/s. That is 3.0×. It is the best single result of the study so far. The Intern is very good at Python. It is still bad at prose, a note that has been in its file since the first day.

![Bar chart of decode speed with the Intern at K = 4, per prompt, against plain decode of about 7 tok/s: the Q4_0 Target is faster on code and prose, the Q4_K_M Target on the structured list]({{ '/assets/charts/p9-target-quant.png' | relative_url }})
*Same Intern, two Targets, four drafted tokens. Q4_0 takes code and prose. The list goes to Q4_K_M, for a reason explained below that is sillier than it looks.*

**Round 2: reading.** Prefill is where the model reads your prompt before it writes a word. It is limited by the math kernels, not by the Bouncer, so it is a different sport. An open llama.cpp PR, [#27851](https://github.com/ggml-org/llama.cpp/pull/27851), adds tiled matrix-multiply kernels for the K-quants. It ships with a runtime switch, so we could A/B it inside one binary. We ran 7B models, three reps per cell, flipping the switch back to back for each model.

Our first pass was spoiled by a stray benchmark loop, orphaned by an earlier run and still going. For part of the pass it benchmarked alongside us, the one kind of help a benchmark cannot use. We killed it and ran everything again.

7B Q4_K_M prefill went from 162 tok/s to 204.
That is +26%.
(Stock measured 155 on the first day. Different session, inside the day-to-day noise.)
7B Q6_K went from 110 tok/s to 391.
That is 3.6×.
Q4_0 did not move.
Decode did not move for any quant.

That left a puzzle. A Q4_K_M file is mostly Q4_K tensors, yet it gained a quarter while pure Q6_K ran off into the distance. So we tried the flag nobody would guess, `--no-repack`.

With the PR and `--no-repack`, 7B Q4_K_M prefilled at about 405 tok/s.
That is 2.5× stock.
7B Q4_0 prefills at 217–243 tok/s on this machine.
So on 7B, the quant that lost Round 1 now reads about 1.7× faster than the one that won it. The 14B Target never ran with `--no-repack`, so the heavyweight rematch is still unbooked.

## Why

**Round 1, measured.** Decode on this machine is bytes per token. Q4_0 is a slightly smaller file, so every token comes out a little sooner. The tied ratio is what the curve [we measured on the first day]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}) predicts: a batch of four tokens costs barely more than one. Q4_0 paid 1.07×. Q4_K_M paid 1.08×. Strictly, a K = 4 step checks five tokens. Four are guesses; one is the Target's own. We measured batches of four, on 7B, and five is in the rerun queue.

**Eight guesses, partly guessed.** At K = 8, a Q4_0 step cost less than a Q4_K_M step with the same Intern work. Q4_0 also had the flattest curve at 16 tokens. Our guess, not profiled: every Q4_0 tensor has a repacked AVX-512 batch kernel, so extra guesses are cheaper to check. But on code the Intern also agreed more often with the Q4_0 Target. We have not separated the two. The 3.0× is part cheap checking, part an Intern who happens to think like Q4_0. Averaged over all three prompts, eight guesses beat four for neither Target.

**The list mystery, measured.** The two Targets do not write the same list. Different quantizations of one model walk different greedy paths, and the Intern guessed the Q4_K_M list and the Q4_0 code better. Per-prompt results swing; the mean holds. Your prompts will pick their own winner.

**Round 2, measured and read in the code.** When llama.cpp loads a Q4_K_M model, it repacks the Q4_K weights into an interleaved layout for an older batched kernel. The dispatcher offers each repacked tensor to that kernel first, so it never reaches the tiled path. By default the PR only gets the Q6_K minority of the file, which fits the small Q4_K_M gain and the big Q6_K one. `--no-repack` skips the repack, and every K-quant tensor goes to the new kernel. The new hire was excellent. It had just been seated behind someone with seniority.

Q4_0 is not a tiled type, and its repacked kernel was already the fast path. Decode pushes one token at a time, and the tiled kernel only takes batches. By the PR's own size threshold, a K = 4 verification step is too small, so we do not expect speculative decoding to gain. That is from reading the code, not from a run. Longer drafts on Q4_K_M with `--no-repack` might clear the bar. Also not run.

The PR's own tests passed. So did all 1323 CPU matmul backend tests. The compiler was the same gcc 14 that gave Q4_0 its crash. Nothing segfaulted, and we checked twice, emotionally.

## What you should do about it

- **Writing lots of text, or running speculative decoding: make the Target Q4_0.** Same speedup ratio, more tokens per second. Draft four tokens by default and eight when the output is code. On Windows with gcc/mingw, check that Q4_0 does not crash on your build first; the post linked at the top has the workaround.
- **Feeding it long prompts (documents, RAG, whole files): Q4_K_M with PR #27851 and `--no-repack`.** That was 2.5× faster prefill on 7B here. The PR is still open, so this means building a branch.
- **Do not use `--no-repack` without the PR.** The repack is doing real work in a stock build. Switching it off without the tiled kernel made prefill slower, not faster.
- **On Q6_K, the PR alone did it, with default flags.** We did not try `--no-repack` there. None of this changed decode, which is still a bytes problem.

In [the next post]({{ site.baseurl }}{% post_url 2026-09-22-three-reviewers-roasted-us %}), three reviewers read everything we had written. They found blockers, plural.

## The receipt

```receipt
GPU POOR LABS                         2026-09-22
------------------------------------------------
ROUND 1: WRITING  14B Target + 0.5B Intern
Q4_0 Target, K=4, mean .... 15.7 tok/s [F-P9-1]
  vs its plain decode .......... 2.22x [F-P9-1]
Q4_K_M Target, K=4, mean .. 14.5 tok/s [F-P9-1]
  vs its plain decode .......... 2.19x [F-P9-1]
  15.7 / 14.5 .................. 1.08x [calc]
Verify 4 vs 1, 7B Q4_0 ......... 1.07x [F-P2-1]
Verify 4 vs 1, 7B Q4_K_M ....... 1.08x [F-P2-1]
  (K=4 checks 5 tokens; 5 not yet run)
Flattest at 16 tokens ........... Q4_0 [F-P2-2]
Q4_0, code, K=8 ........... 21.3 tok/s [F-P9-2]
  plain decode ............. 7.0 tok/s [F-P9-2]
  speedup, best so far .......... 3.0x [F-P9-2]
  (cheap checks vs acceptance: not split)
------------------------------------------------
ROUND 2: READING  7B prefill, PR #27851
Q4_K_M, stock .............. 162 tok/s [F-P11-1]
  first-day stock run ....... 155 tok/s [F-P1-6]
Q4_K_M, PR ................. 204 tok/s [F-P11-1]
  204 / 162 .................... 1.26x [calc]
Q4_K_M, PR + --no-repack .. ~405 tok/s [F-P11-1]
  vs stock ...................... 2.5x [F-P11-1]
Q6_K, stock -> PR ......... 110 -> 391 [F-P11-2]
  gain .......................... 3.6x [F-P11-2]
7B Q4_0 prefill .............. 217-243 [F-P7-4]
  405 / 243 ..................... 1.7x [calc]
Q4_0 with PR ............... unchanged [F-P11-3]
Decode, every quant ........ unchanged [F-P11-3]
CPU matmul tests, gcc 14 ... 1323/1323 [F-P11-4]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae + P6/P7 patches
          round 2 adds PR #27851 @ 190d00d
RUNS      R1: 3 prompts x 2 reps, 16 threads,
          back to back in one session (first
          session binned: machine borrowed)
          R2: 3 reps, switch on/off back to
          back per model; --no-repack: 2 runs
          (batched-bench); light background
          load on the machine during R2
PENDING   verify 5 vs 1: rerun queued
          14B + --no-repack: not run
          longer drafts + --no-repack: not run
THANK YOU FOR NOT BUYING A GPU
```
