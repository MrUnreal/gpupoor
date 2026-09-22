---
layout: post
title: "Your 16-core CPU writes at 98% speed with half its cores asleep"
date: 2026-09-20 11:00:00 -0400
sticker: "98%"
sticker_note: "of full speed, using half the cores"
tags: [bandwidth, decode, prefill, threads, quantization]
lab: P1
summary: "Twelve dense models, one law: on our desktop CPU, text generation runs at 90–99% of what the RAM measurably delivers, and 8 of 16 cores already get 98% of the speed."
---

We own a 16-core CPU. We sent eight of the cores home. Text generation got 2% slower. The Bouncer did not even look up.

## What we measured

[The first post]({{ site.baseurl }}{% post_url 2026-09-20-we-are-gpu-poor %}) made one claim. Tokens per second should be the Bouncer's rate divided by the bytes read per token. The Bouncer is the door between our cores and our RAM, and it lets weights in at 57 GB/s. Nothing else should matter. Not the model family, not the quant, not how clever the code is.

Claims are cheap, so we lined up 12 dense models. They run from 0.5B to 14B parameters. Their quants run from Q2_K to F16. Each one went through `llama-bench` on 16 threads, three reps, median. Then we multiplied tokens per second by bytes per token. That gives the rate at which each model actually pulled weights out of RAM.

![Bar chart of weight bytes streamed per second during decode for 13 models. Twelve dense models sit just under the measured 57 GB/s ceiling; one MoE model falls well short.]({{ '/assets/charts/p1-bandwidth-line.png' | relative_url }})
*Twelve dense models, one wall. The orange bar is a mixture-of-experts model that did not get the memo. [It gets its own post]({{ site.baseurl }}{% post_url 2026-09-20-thirty-five-billion-three-active %}).*

Every dense model decoded at 90–99% of the measured ceiling. (Give or take 5%, depending on which STREAM run we call the ceiling.) Tiny or large, 2-bit or 16-bit, they all queue at the same door. The Bouncer does not check what you are wearing. Our Target, Qwen2.5-14B at Q4_K_M, is on the line too. It writes at 6.3–6.7 tok/s, depending on the session.

Quantization obeys the same law, to within a few percent. The first post ran Qwen2.5-0.5B at F16, Q8_0 and Q4_K_M. Here are its numbers as ratios.

Going from F16 to Q8_0 cuts the bytes by 1.87×. Speed went up 1.88×. Going from Q8_0 to Q4_K_M cuts the bytes by 1.36×. Speed went up 1.33×. Quantization is a diet, not a turbo. You get exactly the weight loss you paid for.

## Why

Decode at batch size one does almost no arithmetic per byte. Each weight is fetched, used for one multiply-add and dropped until the next token. So the real question is how many cores it takes to keep the Bouncer busy. **Measured:** we swept it on Qwen2.5-7B at Q4_K_M, decode only.

| cores | tok/s | share of 16-core speed |
|---:|---:|---:|
| 1 | 7.7 | 60% |
| 2 | 10.8 | 84% |
| 4 | 11.7 | 91% |
| 8 | 12.6 | 98% |
| 16 | 12.9 | 100% |

One core is the interesting row. STREAM is a plain memory-reading benchmark. From a single core it pulls 52–54 GB/s. So the Bouncer would let one core carry out nearly the whole allowance. But one core decoding that 7B model streams only 34 GB/s of weights. The door is open. The core is still tying its shoes.

**Measured:** that gap is the kernel, not the RAM. **Guessed:** the shoelaces are the unpacking. Each 4-bit weight has to be unpacked, scaled and multiplied, and that costs instructions per byte. From reading the code, the quantized decode kernels for x86 in this llama.cpp build are 256-bit AVX2 code, and this chip has AVX-512. We think that is part of the story. We have not tested it.

Two cores close most of the gap. Four close nearly half of what is left. By eight cores the Bouncer is the limit. Cores nine to sixteen add about 2%. They are extra movers hired for a flat with one narrow door. Ask for 16 threads and all sixteen show up for work. Most of that work is standing in the Bouncer's queue. (That last part is inferred from the numbers. We did not profile it.)

**Prefill is a different sport.** Reading a prompt processes a whole batch of tokens per weight read. Each byte the Bouncer lets in now feeds many multiply-adds, so the kernel sets the pace, not the door. Qwen2.5-7B at Q4_K_M prefills at 155 tok/s. That works out to about 2.2 TFLOP/s-equivalent. This chip's AVX-512 FP32 peak is about 5 TFLOP/s. So prefill runs at roughly 40% of peak. The kernel actually does int8 math, whose peak is higher still, so 40% is the generous reading. It is kernel-bound, with headroom. We think every core earns its salary here; we have not swept prefill threads to prove it. (Our notes first called this "near peak". It was not near peak, and [how we found out]({{ site.baseurl }}{% post_url 2026-09-22-three-reviewers-roasted-us %}) is its own post.)

## What you should do about it

- **Predict before you download,** with [the first post's rule of thumb]({{ site.baseurl }}{% post_url 2026-09-20-we-are-gpu-poor %}): measured GB/s ÷ GB read per token. Twelve dense models delivered 90–99% of that prediction.
- **Choose the quant by its bytes.** For generation, a quant is exactly as fast as it is small. Pick the smallest one whose quality you can live with, and do not expect a speed bonus on top.
- **Try 8 threads for generation.** On the 7B it cost about 2%. It hands back eight cores. llama.cpp takes separate thread counts for generation (`-t`) and prompt processing (`-tb`), so `-t 8 -tb 16` should keep every core on the prompt. We have not measured that pairing. It is a guess built on the decode sweep and the prefill arithmetic above.
- **Count physical cores, not SMT threads.** Sixteen is the ceiling here, not thirty-two. What happens at thirty-two is a horror story for [the next post]({{ site.baseurl }}{% post_url 2026-09-20-openmp-tax %}).

So decode uses eight cores and a rounding error. The other eight are paid for and have nothing useful to do. We suspect even the busy ones spend most of their time waiting on the Bouncer. The rest of this series is about finding those idle cores a job: [fetching scattered rows]({{ site.baseurl }}{% post_url 2026-09-20-sparse-is-free-with-friends %}), [checking several answers in one trip]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}), then [hiring an Intern]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}).

## The receipt

```receipt
GPU POOR LABS                         2026-09-20
------------------------------------------------
Measured RAM ceiling .......... 57 GB/s [F-P0-2]
Dense models tested ................ 12 [F-P1-1]
Their share of ceiling ......... 90-99% [F-P1-1]
  (+/-5% by which STREAM cell is used)
14B Q4_K_M decode ....... 6.3-6.7 tok/s [F-P1-3]
  (varies by session)
0.5B F16     0.99 GB/tok   54.8 tok/s   [F-P1-2]
0.5B Q8_0    0.53 GB/tok  102.9 tok/s   [F-P1-2]
0.5B Q4_K_M  0.39 GB/tok  136.7 tok/s   [F-P1-2]
  F16->Q8_0  bytes 1.87x, speed 1.88x    derived
  Q8_0->Q4K  bytes 1.36x, speed 1.33x    derived
7B Q4_K_M decode tok/s by cores:        [F-P1-4]
  1: 7.7  2: 10.8  4: 11.7  8: 12.6  16: 12.9
  1/2/4/8 vs 16 .......... 60/84/91/98%  derived
  cores 9-16 add .................. ~2%  derived
1 core, STREAM read ........ 52-54 GB/s [F-P0-3]
1 core, 7B Q4_K_M decode ...... 34 GB/s [F-P1-5]
7B Q4_K_M prefill ........... 155 tok/s [F-P1-6]
  TFLOP/s-equivalent ............. ~2.2 [F-P1-6]
  AVX-512 FP32 peak, TFLOP/s ....... ~5 [F-P1-6]
  share of peak .................. ~40% [F-P1-6]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae, gcc 14, no OpenMP
TEST      llama-bench pp512 / tg128
RUNS      models: 16 threads, 3 reps, median
          cores: one sweep, llama-bench mean
NOT RUN   prefill thread sweep; -t 8 -tb 16
GUESSED   why 1 core is slow (AVX2, unpacking)
          what busy threads wait on (no profile)
THANK YOU FOR NOT BUYING A GPU
```
