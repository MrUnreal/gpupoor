---
layout: post
title: "We interviewed six interns. The cheapest one made us 2× faster."
date: 2026-09-20 20:00:00 -0400
sticker: "2.0×"
sticker_note: "faster, every word still the Target's call"
tags: [speculative-decoding, draft-models]
lab: P3
summary: "A 0.5B model guessing four tokens ahead makes a 14B model write 2.0–2.2× faster on a desktop CPU; bigger interns did worse and higher precision bought nothing."
---

The Target writes 6.3 to 6.7 tokens per second on our CPU. Each token costs one full read of its weights from RAM. The Bouncer lets in 57 GB/s. That is the whole speed limit.

Earlier that day we found a loophole: [checking four tokens costs about the same as writing one]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}). So we posted a job ad. *Wanted: one Intern to guess the Target's next few words, so the Target only has to check them. Pay: none. Perks: leftover bandwidth.*

Six candidates applied. We hired the cheapest. It doubled our speed.

## What we measured

This is speculative decoding in llama-server. The Intern drafts K tokens. The Target checks all of them in one batched pass. It keeps the longest run it agrees with, then adds one token of its own. Decoding is greedy, and the Target is Qwen2.5-14B at Q4_K_M.

The applicants were all Qwen2.5, the Target's own family, so they share its vocabulary. Three were the 0.5B model, at Q4_K_M, Q8_0 and F16. Two were the 1.5B, at Q4_K_M and Q8_0. One was the 3B, at Q4_K_M. Each interviewed on the same three prompts: write a Python module, explain in prose why CPUs are slow at this (the Target had opinions), and list chemical elements with a fact about each.

![Bar chart of decode speedup at K = 4 for six draft models. Qwen2.5-0.5B at Q4_K_M and Q8_0 tie at 2.02×; the 3B draft comes last at 1.39×.]({{ '/assets/charts/p3-draft-size.png' | relative_url }})
*The interview results at K = 4. Sorted by weight, and also by speed. It is the same order.*

The two smallest Interns tied at 2.02×. The 3B came last at 1.39×. Across two sessions, the winner's mean lands at 2.0–2.2×.

The 3B Intern agreed with the Target more often than any other candidate. It was also the worst at the job. The F16 Intern is our favourite failure. It reads 0.99 GB per guess. Its Q4_K_M twin reads 0.39 GB. The F16 twin's guesses were accepted 53% of the time. The Q4_K_M twin's: 51%. The Q8_0 twin's: 55%. Nobody asked the Intern for decimal places.

Then we asked each Intern for more guesses per round.

![Line chart of speedup against draft length K for the 0.5B, 1.5B and 3B drafts at Q4_K_M. Every line peaks at K = 4 and falls at K = 16.]({{ '/assets/charts/p3-draft-length.png' | relative_url }})
*Every Intern peaks at four guesses per round. Sixteen never beats two.*

K = 4 won for every Intern. Asking for sixteen is asking the Intern to finish the Target's paragraph. It confidently writes one. The Target throws most of it away.

The Intern is also a coder, not a novelist. With the best setup, code ran ~2.5× faster. The list ran ~2.1× faster. Prose ran ~1.5× faster.

### The applicants we did not hire

One applicant had no brain at all. N-gram self-speculation drafts by looking up what followed the same few tokens earlier in the text. It is a photocopier with ambition. On freshly generated text there was almost nothing to copy. It scored 0.95–1.01×. That is noise wearing a lanyard.

Another came with the building. Qwen3.6-35B-A3B, a mixture-of-experts model, ships with its own multi-token prediction head: one small extra layer trained to guess ahead. No second model to find, load or feed. At K = 2 it took the MoE from 19.4 to 25.4 tokens per second. That is 1.31×, for free. The MoE [has other problems]({{ site.baseurl }}{% post_url 2026-09-20-thirty-five-billion-three-active %}), but its intern is not one of them.

## Why

**Measured: the check is cheap.** Verifying 4 tokens costs 1.07–1.14× the price of one. That was one run per point, on a benchmark that stands in for the real check, with a rerun queued. So every guess the Target accepts is almost free output. That is the whole business model.

**Measured: the Intern is not free.** It drafts one token at a time while the Target waits, and each drafted token is a read of the Intern's own weights. The F16 twin guessed as well as its Q4_K_M sibling and still finished well behind it. It also reads 2.5× the bytes per guess. We blame the bytes.

**Our explanation, which fits the grid but was not measured on its own:** a bigger Intern agrees more, but its cost per guess grows faster than its agreement. The chart has a built-in control. The 1.5B at Q4_K_M and the 0.5B at F16 weigh the same per guess. The one with more brains edged out the one with more decimal places. That is one run each, so call it a lean, not a verdict.

**Measured: precision does not buy agreement.** That is the 51 / 55 / 53% above. **Our guess why:** the Intern's mistakes come from what it knows, not from how precisely it stores it. Rounding a small brain harder costs far less than being a small brain.

**Measured, then guessed: long drafts fail twice.** Checking 16 tokens costs 1.48–1.74× one token. So the check stops being free. Meanwhile each extra guess is a longer bet on the Intern being right all the way down, and the chart's footnote shows per-token acceptance sliding as K grows. Four is where the check is still nearly free and the bet is still worth taking.

**Measured: the Intern does worst on prose. Guess why: code has fewer right answers.** After `for i in`, nearly everyone says `range`. In prose, if the Target would write "bottleneck" and the Intern offers "limit", the guess is gone. A synonym is a rejection.

### Same answers?

In greedy speculative decoding, every token that survives is one the Target picked itself. So the output should match plain decoding byte for byte. It does not. Neither does plain decoding at 8 threads versus 16.

The likely cause, from the lab notes: a different thread count changes the order of additions, and the batched check runs different kernels from one-token decoding. Near-tied choices flip. So the sticker promises the Target's picks, not the same bytes. String equality cannot referee this. A proper token-level score is being redone, and until it lands we claim nothing more.

## What you should do about it

- **Hire the smallest model in your Target's family, at Q4_K_M or Q8_0.** For Qwen2.5-14B that is Qwen2.5-0.5B: `-md qwen2.5-0.5b-instruct-q4_k_m.gguf`. Skip F16. Skip the bigger siblings.
- **Draft four tokens.** That is `--spec-draft-n-max 4` in the llama.cpp build we used. Flag names move between versions, so check `--help`.
- **Tell idle threads to sleep.** Add `--poll 0 --spec-draft-poll 0`. The whole interview grid ran with those flags. Without them, the Spinners eat a big slice of the gain, and [that is a whodunit of its own]({{ site.baseurl }}{% post_url 2026-09-21-we-paid-sixteen-threads-to-spin %}).
- **If your model came with its own intern, use it.** `--spec-type draft-mtp --spec-draft-n-max 2` on a model with an MTP head. Do not expect n-gram lookup to help on fresh text.

## The receipt

```receipt
GPU POOR LABS                         2026-09-20
------------------------------------------------
Target 14B Q4_K_M, plain .. 6.3-6.7 t/s [F-P1-3]
Bouncer (STREAM read) ......... 57 GB/s [F-P0-2]
Hired Intern, mean, K=4 ...... 2.0-2.2x [F-P3-1]
  2.02x one session, 2.19x another
Code/list/prose ....... ~2.5/~2.1/~1.5x [F-P3-2]
INTERVIEWS AT K=4                       [F-P3-3]
  0.5B Q4_K_M ................... 2.02x
  0.5B Q8_0 ..................... 2.02x
  1.5B Q4_K_M ................... 1.80x
  0.5B F16 ...................... 1.70x
  1.5B Q8_0 ..................... 1.47x
  3B Q4_K_M ..................... 1.39x
Accepted Q4_K_M/Q8_0/F16 .... 51/55/53% [F-P3-4]
GB per guess Q4_K_M/F16 ..... 0.39/0.99 [F-P1-2]
  F16 bytes, 0.99/0.39 ........... 2.5x
Best draft length ................. K=4 [F-P3-5]
  K=16 never beats K=2
Verify 4 / verify 1 ........ 1.07-1.14x [F-P2-1]
Verify 16 / verify 1 ....... 1.48-1.74x [F-P2-2]
  stand-in benchmark, 1 run per point
N-gram, fresh text ......... 0.95-1.01x [F-P3-6]
MTP head, 35B-A3B, K=2 ..... 19.4->25.4 [F-P3-7]
  tok/s, speedup ................ 1.31x
Output byte-identical ............... no [F-Q-1]
  nor is plain at 8 vs 16 threads
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae, gcc 14, OpenMP off
FLAGS     grid: --poll 0 --spec-draft-poll 0
          2.19x: patched build, pools parked
RUNS      grid 1 per cell; 2.19x 2 reps;
          P2 1 per point
PENDING   token-level quality score (v2),
          P2 rerun at 3 reps
THANK YOU FOR NOT BUYING A GPU
```
