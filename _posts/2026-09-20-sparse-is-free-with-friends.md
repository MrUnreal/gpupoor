---
layout: post
title: "Skipping 99% of the rows is 72× faster, if you bring 15 friends"
date: 2026-09-20 15:00:00 -0400
sticker: "72×"
sticker_note: "speedup reading 1% of rows, ideal 100×"
tags: [sparsity, memory-bandwidth, threads]
lab: P4
summary: "The textbook says scattered reads waste memory bandwidth. One core agrees. In our synthetic test, sixteen cores read a random 1% of the rows 72× faster than reading all of them (ideal: 100×)."
---

Before we ran a single benchmark, we wrote ourselves a research brief. One of its sections is titled "Why Sparsity Has Not Solved It". Its first bullet says that gathering scattered rows trades bandwidth for latency, and that streaming everything in order is vastly more efficient. We tested that bullet on the first day. It turned out to be true for one core and mostly false for sixteen.

## What we measured

The idea on trial is neuron sparsity. For each token, a small predictor guesses which rows of a weight matrix will actually matter, and you read only those. Read 1% of the rows and, on paper, you finish 100× sooner. The textbook says you will not, because random jumps defeat the hardware's read-ahead and every jump is a fresh trip to RAM.

So we built the smallest honest version of the fight. One matrix of 1 GiB, too big for any cache on the chip, the VIP Lounge included. The dense run reads every row in order. The sparse run reads a random subset of rows, picked fresh every pass, the way a predictor would pick them. Then we swept how many rows get read, how big each row is, and how many threads do the reading.

![Log-scale chart of speedup over reading every row versus the fraction of 1 KiB rows read. Sixteen threads run close to the ideal 1/density line and reach 72× at 1%. One thread reaches 35× at 1% and drops below 1× at 50%.]({{ '/assets/charts/p4-gather-vs-stream.png' | relative_url }})
*The thin grey line is the ideal, 1/density. Sixteen threads live next door to it. One thread lives downstairs, and at 50% it falls through the floor.*

One thread behaved exactly as warned. We asked it to read half of the 1 KiB rows, in random order. It was slower than reading all of them.

The speedup was 0.8×.

It skipped half the work and arrived late.

Sixteen threads read the same brief and ignored it. For rows of 1 KiB and up, from half the rows down to a tenth, they landed at 85–95% of the ideal speedup. At 1% of the rows the ideal is 100×. Sixteen threads got 72×. Four threads got 66×. One thread got 35×.

## Why

Start with what we had already measured. One core reading in a straight line moves 52–54 GB/s. The Bouncer lets in 57. For a plain in-order read, one core nearly fills the doorway by itself. In dense decode, extra cores stop helping long before you run out of them. That was the whole point of [the post about the sleeping cores]({{ site.baseurl }}{% post_url 2026-09-20-your-cpu-is-asleep %}).

Now the explanation. It is the standard one and it fits the data, but we did not measure it directly: we never counted requests in flight. When a core reads in a straight line, the hardware prefetcher spots the pattern and fetches ahead. When the next row lives somewhere random, the prefetcher has nothing to go on. The core asks for a row, waits, gets it, and asks again. It can only keep a handful of requests in flight at once. So one core spends most of a sparse read waiting, not receiving.

Sixteen cores each keep their own handful in flight. The waits overlap. The Bouncer stops seeing one guest at a time and starts seeing a queue. A steady queue is all the Bouncer ever wanted. The architecture term is memory-level parallelism. The GPU-poor term is bringing friends. The latency never went away. It got split sixteen ways, like a dinner bill.

We think row size matters for the same reason: the reads inside a row are in order again, so each jump comes with a little runway. At 1 KiB there is enough runway. Rows of 256 bytes are four cache lines long. That is barely a runway. They lose 25–35%, even with all sixteen threads.

We also tried helping. The kernel told the CPU which row it would want a few rows from now. With sixteen threads, software prefetch had no measurable effect. We think the queue at the door was already full, so a tip about the next guest got nobody in sooner. The Bouncer pocketed the tip and let in exactly as many bytes as before.

The shortfall at 1% is our least certain part. There, each pass is so short that we think the fixed cost of waking sixteen OpenMP threads becomes a visible slice of it. That is a guess, not a measurement, though OpenMP on this machine [has a record]({{ site.baseurl }}{% post_url 2026-09-20-openmp-tax %}).

Now the fine print, which is load-bearing. This is a synthetic kernel. It sums bytes and does no dequantization math, so it measures the memory side only. It has no predictor, and the predictor's cost is real and unmeasured. And this is a desktop with two memory channels. A server with many more channels should need more requests in flight to fill. There, we expect our one- and four-thread lines to be the better guide.

Even with that fine print, the result changes how we read the neuron-sparsity papers, Deja Vu and PowerInfer among them. They got their big wins on ReLU models, where most FFN neurons output exactly zero for a given token. A paper even gave it a name: the Lazy Neuron Phenomenon. But the model families people actually run use SwiGLU, which leaves far fewer clean zeros. The neurons got day jobs.

Our brief listed the gather as one reason the idea stalled. On this desktop, with every core pitching in, the gather keeps most of the win. Our reading of the literature, which is an opinion and not a measurement, is that the missing ingredient is the models.

## What you should do about it

There is no llama.cpp flag for this. Sorry. These are for anyone building, or judging, a sparse method on a CPU.

- **Benchmark it with every core.** A one-thread microbenchmark will show you 35× where 100× was promised. It is only telling you about one core.
- **Keep the thing you skip at 1 KiB or bigger.** Quantized FFN rows in the 7B and 14B models we run already clear that. The trap is the down-projection: its neurons are columns, and skipping a column saves nothing unless the matrix is stored transposed, as PowerInfer does.
- **Do not hand-write prefetches at 16 threads.** In our kernel they did nothing there. Spend that effort on the predictor, the part nobody here has measured yet.
- **If you want a project:** take a ReLU-sparse model (ProSparse or TurboSparse style) in GGUF and add a row-skip to the CPU FFN path. On a 16-core desktop like ours, the memory side should not be what stops you.

Next, the idle cores get a second job: [checking four answers for about the price of one]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}).

## The receipt

```receipt
GPU POOR LABS                         2026-09-20
------------------------------------------------
1% of 1KiB rows, 16 threads ...... 72x [F-P4-2]
1% of 1KiB rows, 4 threads ....... 66x [F-P4-2]
1% of 1KiB rows, 1 thread ........ 35x [F-P4-2]
Ideal at 1% (1/density) ......... 100x [F-P4-2]
16 thr, >=1KiB rows, 10-50% dens
  share of ideal .............. 85-95% [F-P4-1]
1 thread, 50% of 1KiB rows ...... 0.8x [F-P4-3]
256B rows at 16 thr, loss ..... 25-35% [F-P4-5]
Software prefetch, 16 thr .. no effect [F-P4-4]
One core, reading in order . 52-54GB/s [F-P0-3]
The Bouncer's ceiling ......... 57GB/s [F-P0-2]
------------------------------------------------
KERNEL    synthetic: sums bytes, no dequant
MATRIX    1 GiB, random rows, fresh each pass
PREDICTOR not modelled, cost unknown
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     standalone C + OpenMP, gcc 14
RUNS      best of 3-5 reps per point
THANK YOU FOR NOT BUYING A GPU
```
