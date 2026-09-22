---
layout: post
title: "35 billion parameters, 3 billion awake, 28% slower than the math"
date: 2026-09-20 22:00:00 -0400
sticker: "28%"
sticker_note: "slower than it should be (we think we know why)"
tags: [moe, bytes-per-token, hybrid-models, corrections]
lab: P5
summary: "A 35B mixture-of-experts with 3B active should decode like a dense 3B on a CPU. It runs 28% slower, and taking threads away barely moves it. Our best guess, not yet proven: a recurrent state nobody put in the byte count."
---

A mixture-of-experts model is a company with 35 billion employees. For any given token, only 3 billion of them come in to work. On a CPU that is the dream. Our one equation says decode speed is bandwidth divided by bytes per token, and the Bouncer only charges for whoever walks through the door.

So Qwen3.6-35B-A3B should decode like a dense 3B model reading the same bytes. It runs 28% slower. Our first explanation lasted until the first experiment built to test it.

## What we measured

The tool was `llama-bench`, decode test `tg128`, 16 threads, everything at Q4_K_M. The MoE went up against two dense 3B models (Qwen2.5-3B and Llama-3.2-3B) that read about the same weight bytes per token.

| model | decode, 16 threads |
|---|---|
| Qwen3.6-35B-A3B (MoE, 3B active) | 19.6 tok/s |
| dense 3B, same weight bytes (two models) | 27.3–27.7 tok/s |

Every dense model we have run sits on the bandwidth line. The MoE is the first model to fall off it.

We did not see that coming. [Earlier the same day]({{ site.baseurl }}{% post_url 2026-09-20-sparse-is-free-with-friends %}) we showed that grabbing scattered rows of memory is nearly free when 16 cores do it together. An expert is a big contiguous block, which is the easy case. So, we reasoned, the gap had to be overhead.

**Theory one: too many meetings.** The MoE runs a pile of small matrix multiplies every token: one per chosen expert, per projection, per layer. ggml splits each of them across all 16 threads. Then every thread waits at a barrier, the meeting from [the OpenMP post]({{ site.baseurl }}{% post_url 2026-09-20-openmp-tax %}), before the next one starts.

The MoE spends about 51 ms on a token. The dense 3B spends 36–37 ms. That is about 15 ms extra per token. Divide it over the pile of tiny jobs. What came out looked just like the price of making 16 threads start and stop in unison, over and over. We wrote it up. We were quite pleased.

A barrier theory makes a prediction. Fewer threads at the barrier means shorter waits. The MoE should speed up as we take threads away.

![Decode speed against thread count: the MoE stays between 20.2 and 21.2 tok/s from 4 to 16 threads, below the dense 3B at every count]({{ '/assets/charts/p5-moe-threads.png' | relative_url }})
*About the same weight bytes per token, very different results. The MoE (orange) barely notices how many threads it gets. The chart's title is our day-one wording: what stays flat is the MoE's own speed.*

It barely moved. The MoE ran 20.8, 21.2 and 20.2 tok/s on 4, 8 and 16 threads. Cutting from 16 threads to 8 bought 1.0 tok/s. Four threads kept up with sixteen.

A barrier tax that barely notices how many threads are at the barrier is not much of a barrier tax. We downgraded the theory the same day. Dispatch cost may still be a bit player. It is not the lead.

## Why: the luggage nobody counted

Qwen3.6-35B-A3B is not just an MoE. It is a hybrid. Of its 41 layers, 30 are gated-delta layers. That is a recurrent design that keeps a fixed-size running state instead of a growing attention cache.

We knew that from the start. After the thread test, those layers even topped our suspect list, as a swarm of small ops. What we missed is that they carry luggage.

Each of those states is 2 MiB of F32. Every token, each layer reads its state, updates it and writes it back. Across the 30 layers, that is 60 MiB of state per token.

Our bytes-per-token math counted weights. The state is not a weight. The guest list said 3 billion, and nobody wrote down the luggage. The Bouncer weighs luggage anyway.

It probably gets worse. By one reviewer's reading of llama.cpp's graph, the state goes through several passes (gather, scan, scatter), not one. Every pass is another round trip past the Bouncer. The dense 3B models have no such layers, so they travel with carry-on only.

We never did that sum. That reviewer, one of three, did it two days later, in what became [the roast]({{ site.baseurl }}{% post_url 2026-09-22-three-reviewers-roasted-us %}). Their estimate: count the state traffic and most of the 28% stops being a tax. It turns into bytes we forgot to put on the bill.

That estimate leans on the extra passes. One read and one write of 60 MiB would cover only a slice of the gap. Nobody has counted the passes on a running model yet.

The labels, plainly:

- **Read off the model file:** 30 of 41 layers carry a 2 MiB recurrent state.
- **Measured:** the 28% gap, and an MoE that barely notices its thread count.
- **Estimated (by a reviewer):** how many passes llama.cpp makes over the state, and so how many bytes it really moves.
- **Guessed:** whatever the luggage does not cover. Our money is on per-op overhead and the single-row activation quantizer, which is plain scalar code on x86.
- **Not run:** the per-op profile that would settle it.

One more confession. The dense 3B models are not a matched control. They come from other families with ordinary attention. So the headline gap mixes two differences at once: experts and hybrid layers. This test cannot split them.

There is good news, and it came in the box: the model's own MTP head, already [interviewed alongside the Intern]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}). It took decode from 19.4 to 25.4 tok/s. That is 1.31×.

## What you should do about it

- **Count the luggage.** If a model's GGUF header has `ssm.*` keys, it carries recurrent state. Multiply out `ssm.state_size × ssm.inner_size` in F32, times the number of such layers. Add that to bytes per token at least twice: once read, once written. The parameter count will not warn you.
- **Do not throw threads at an MoE gap.** Ours ran at least as fast on 8 threads as on 16. If you want cores back for something else, take them from the MoE.
- **Budget RAM for 35 billion, bandwidth for 3 billion plus luggage.** The sleeping experts cost you capacity, not speed. The awake ones and their state are what the Bouncer bills.

## The receipt

```receipt
GPU POOR LABS                        2026-09-20
------------------------------------------------
MoE 35B-A3B, 16 thr .... 19.6 t/s     [F-P5-1]
Dense 3B, same bytes ... 27.3-27.7    [F-P5-1]
Gap vs dense ........... 28%          [F-P5-1]
ms/token = 1000 / (t/s):
  MoE .................. 51           derived
  dense 3B ............. 36-37        derived
  extra per token ...... ~15 ms       derived
MoE @ 4/8/16 thr ....... 20.8/21.2/20.2
                                      [F-P5-2]
16 -> 8 thr, 21.2-20.2 . 1.0 t/s      derived
Layers with state ...... 30 of 41     [F-P5-3]
State per layer, F32 ... 2 MiB        [F-P5-3]
State, 30 x 2 MiB ...... 60 MiB       derived
MTP head, K=2 .......... 19.4->25.4   [F-P3-7]
MTP speedup ............ 1.31x        [F-P3-7]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae, gcc 14, OpenMP off
RUNS      llama-bench tg128, Q4_K_M;
          baseline 3 reps, sweep 2 reps
          MTP: llama-server, 3 prompts,
          1 run each, mean
NOTE      baseline, sweep and MTP are
          separate runs (hence 19.6 at 16
          thr, 20.2 in the sweep, 19.4 MTP)
STATE     leading hypothesis, found 2026-09-22
          pass count estimated
          per-op profile not run
CONTROL   dense 3Bs are other model families
THANK YOU FOR NOT BUYING A GPU
```
