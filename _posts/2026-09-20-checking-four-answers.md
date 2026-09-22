---
layout: post
title: "Checking four answers costs 8% more than checking one"
date: 2026-09-20 18:30:00 -0400
sticker: "1.08×"
sticker_note: "cost of verifying 4 tokens vs 1"
tags: [speculative-decoding, verification]
lab: P2
summary: "On a 16-core desktop CPU, checking four tokens in one pass costs 1.07–1.14× checking one, because all four ride the same trip to RAM."
---

Our Target writes one token by reading every one of its weights out of RAM. Then it reads every one of them again to write the next token. It is the household that drives to the shop once per egg.

Speculative decoding is the plan to buy a carton. A tiny model guesses the next few tokens, and the Target checks all the guesses in one trip. Before hiring anyone to guess, we priced the carton.

## What we measured

We priced the checking without a guesser, using a stand-in. llama.cpp ships `llama-batched-bench`, which runs N sequences side by side, each producing one token per step. For the weights, that is the same matrix work as one sequence checking N guessed tokens. The attention part differs slightly, and at our short prompt it barely registers. Divide the step time at N by the step time at 1, and you have the price of checking N answers.

We ran it on Qwen2.5-7B at Q4_0, Q4_K_M and Q8_0, and on the 14B Target at Q4_K_M. Then we ran the 7B Q4_K_M once more, squeezed onto 8 threads pinned to a single die. Everything else ran on 16 threads. (Those 8 threads turned out to be sitting on only 4 physical cores. See the correction at the bottom.)

![Line chart of step time relative to a one-token step, against tokens checked per step, for Qwen2.5-7B at Q4_0, Q4_K_M and Q8_0 on 16 threads and Q4_K_M on 8 threads pinned to one die. At four tokens the 16-thread curves sit at 1.07 to 1.14 times; the pinned curve bends first.]({{ '/assets/charts/p2-verification-cost.png' | relative_url }})
*Step time relative to a one-token step, 7B only. The grey diagonal is linear cost, the worst case: 4× at four tokens, off the top of the chart soon after. The line labelled "8 thr, one CCD" is really 4 physical cores; see the correction below.*

The price list:

- Checking 4 tokens costs **1.07–1.14×** checking one.
- On Q4_K_M, the Target's own quant, it is **1.08×**.
- Checking 8 tokens costs **1.24–1.35×**.
- Checking 16 tokens costs **1.48–1.74×**.
- With 8 threads pinned to one die, 8 tokens cost ~~1.60× on 8 cores~~ **1.60×**, and 16 tokens cost **2.25×**. Those threads were on 4 physical cores, not 8. Correction pending.

Each point is a single run. A three-rep rerun is queued, so hold the second decimal loosely.

Flip it around and it looks even better. One ordinary step's worth of time buys about **3.7** checked tokens (4 ÷ 1.08).

We went in nervous. The only published measurement of this curve for llama.cpp's quantized kernels that we could find ran on Apple's Metal backend. There, on Q4_K_M, the cost of checking grew roughly in step with the number of tokens ([arXiv 2607.17283](https://arxiv.org/abs/2607.17283)). If x86 behaved like that, speculative decoding on Q4_K_M would lose money before the Intern said a word. Our CPU does not behave like that. Different chips, different kernels, and both measurements can be right.

Our own planning notes predicted the curve would not be flat. It is not flat. It is just much flatter than we feared.

## Why the extra eggs are nearly free

Start with what [an earlier post]({{ site.baseurl }}{% post_url 2026-09-20-your-cpu-is-asleep %}) measured. Decode runs at 90–99% of the RAM's measured ceiling. That ceiling is 57 GB/s. The Bouncer does not negotiate. On the 7B, eight cores already reach 98% of the speed of sixteen.

Now the mechanism. The one-trip part is arithmetic, not a guess. A single token per step already uses most of the ceiling. Four separate trips to RAM could never fit in 1.08× the time.

The rest is the textbook story; our curves fit it, but we did not run a profiler to prove it. At one token per step, most of the CPU is standing in the hallway waiting for bytes. Every weight that comes through the door gets multiplied by every token in the batch. One token means one multiply per weight, then a wait. Four tokens means four multiplies per weight, and the same wait. The extra arithmetic lands on cores that were idle anyway. You pay for the trip to RAM once, and the maths rides along.

**Measured:** the free lunch ends somewhere between 8 and 16 tokens. **Guessed:** past that point the cores stop waiting for RAM and start waiting for themselves. The pinned run fits that story. At one token it is just as fast, because RAM is the limit either way. With less of the chip doing the maths, the bill arrives early. We first wrote "half the chip, half the headroom". It was a quarter of the chip: the 8 threads shared 4 cores. The real half-chip number is being remeasured, and it will matter later, when we try giving the Target one die and the Intern the other.

**Measured:** the 14B Target draws the same shape as the 7B. With two models that is a hint, not a law, but the curve looks like a property of the kernel and the core count rather than of the model.

**Measured:** Q4_0 has the flattest curve at 16 tokens, which is what our planning notes bet on. Q8_0, whose dot product is the simplest of the three, paid the most at four tokens. The simplest arithmetic did not buy the cheapest check.

**Guessed** (from reading the source, not from a profiler): nearly all of a Q4_0 file's weights get repacked for an AVX-512 matrix-multiply path. A Q4_K_M file stores many more of its tensors as Q6_K. Those fall back to a one-row-at-a-time AVX2 kernel, even when there is a whole batch to share. We think that fallback is what bends the Q4_K_M curve.

One asterisk on the Q4_0 line. On our stock gcc build, every Q4_0 model crashed, so its curve comes from our patched build. The crash is [a whole saga of its own]({{ site.baseurl }}{% post_url 2026-09-21-gcc-cannot-count-to-32 %}).

## What you should do about it

- **Try speculative decoding on your CPU.** On this machine the checking is cheap. Checking four tokens costs 7–14% more than checking one.
- **Keep drafts short.** Checking stays cheap up to about 8 tokens and gets pricey after that. On this llama.cpp build the knob is `--spec-draft-n-max`, and the Target checks one token more than the draft length. Past the knee, every extra guess costs real money, including the ones the Target throws away.
- **Give the Target every core while it checks.** Squeezed onto 4 cores, 16 tokens cost 2.25×. The same 7B Q4_K_M on all 16 cores pays 1.74×. (How much of that survives on 8 honest cores: correction pending.)
- **Price your own carton before you go shopping.** One command measures the curve for any model: `llama-batched-bench -m model.gguf -pps -npp 128 -ntg 32 -npl 1,2,4,8,16`. Divide each row's `T_TG s` by the first row's. If those ratios climb anywhere near 2, 4, 8, 16, no Intern can save you.

The carton is cheap. Now we need someone to fill it, so we went and [hired the cheapest Intern we could find]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}).

## The receipt

```receipt
GPU POOR LABS                      2026-09-20
------------------------------------------------
VERIFY N TOKENS / VERIFY 1, 16 THREADS
 4 tok, Q4_0 ................ 1.07x  [F-P2-1]
 4 tok, Q4_K_M (7B, 14B) .... 1.08x  [F-P2-1]
 4 tok, Q8_0 ................ 1.14x  [F-P2-1]
 8 tok, all 4 curves ... 1.24-1.35x  [F-P2-2]
16 tok, all 4 curves ... 1.48-1.74x  [F-P2-2]
16 tok, Q4_0 (flattest) ..... 1.48x  [F-P2-2]
16 tok, 7B Q4_K_M ........... 1.74x  [F-P2-2]
8 THREADS PINNED, REALLY 4 CORES, 7B Q4_K_M
 8 tok ...................... 1.60x  [F-P2-3]
16 tok ...................... 2.25x  [F-P2-3]
 (8 real cores: rerun pending)      [F-PIN-1]
DERIVED
 1.08 - 1 ..................... +8%  [F-P2-1]
 1.07..1.14 - 1 ............ +7-14%  [F-P2-1]
 4 / 1.08, tokens per step .... 3.7  [F-P2-1]
CONTEXT
 Decode vs ceiling ......... 90-99%  [F-P1-1]
 Ceiling (STREAM) ......... 57 GB/s  [F-P0-2]
 8 cores vs 16, 7B ............ 98%  [F-P1-4]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae, gcc 14, no OpenMP
          Q4_0 on our crash-fix branch
MODELS    Qwen2.5-7B x3 quants, 14B Q4_K_M
PROXY     batched-bench stands in for verify
RUNS      1 per point (3-rep rerun queued)
THANK YOU FOR NOT BUYING A GPU
```

## Corrections

**2026-09-22 — the "one die" line was 4 cores, not 8.** A fact-checker reading llama.cpp's thread placement noticed that `--cpu-range 16-31 --cpu-strict 1` hands thread *i* the *i*-th allowed logical CPU, and Windows numbers the two hyperthreads of each core next to each other. So "8 threads on one die" meant cores 8–11, two threads each. The numbers above are what that layout measured; the label was wrong. A rerun on 8 real cores (mask `0x55550000`) is running, and this post will get the corrected curve. Everything measured on 16 unpinned threads is unaffected.
