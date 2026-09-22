---
layout: post
title: "We are GPU poor, and our RAM scored 57 out of 96"
date: 2026-09-20 09:00:00 -0400
sticker: "57 GB/s"
sticker_note: "what the RAM actually delivers"
tags: [manifesto, memory-bandwidth, stream]
lab: P0
summary: "One desktop CPU, no GPU, a spec sheet that promises 96 GB/s of memory bandwidth and a benchmark that finds 57: the speed limit for nearly every model on this blog."
---

We want a 14-billion-parameter language model to write for us on one desktop CPU, with no GPU. Technically there is an RTX 5080 in the case. It is plugged in, it is excluded from every benchmark, and it is not allowed to help. We are GPU poor on purpose, which is the most expensive way to be poor.

## What we measured

The whole theory of this blog fits on one line:

**tokens per second ≈ memory bandwidth ÷ bytes read per token**

To write one token, the model reads nearly every weight it has from RAM. It uses each weight for one multiply-add and forgets it. Then it does the whole thing again for the next token. The arithmetic is cheap and the reading is not, so the speed limit is how fast RAM can hand over bytes.

So on the first day we asked the RAM how fast it is. The spec sheet answered first. Two sticks of DDR5-6000 on two channels move 6000 million transfers a second, 8 bytes each, times two.

That is 96 GB/s.

Then we ran a STREAM-style read test. It reads arrays far too big for any cache and times the trip. Here is what came back:

| | read bandwidth |
|---|---|
| Spec sheet | 96 GB/s |
| Measured peak | **57 GB/s** |
| Measured, one core alone | 52–54 GB/s |

The RAM scored 57 out of 96. That is 59%.

The last row hurt more. One core, on its own, streams 52 to 54 GB/s. Fifteen more cores add a few GB/s on top. This machine has three 96s on its spec sheet: 96 GB of RAM, 96 MB of L3 on its V-cache die and 96 GB/s of bandwidth. Two of them show up for work.

A theory needs a test, so we took the smallest model in our first lineup, Qwen2.5-0.5B, at three precisions. For each one we multiplied its measured speed by the bytes it reads per token. If the equation is right, every row should land on the same bandwidth.

| Qwen2.5-0.5B | GB read per token | tok/s | GB/s implied (≈) |
|---|---|---|---|
| F16 | 0.99 | 54.8 | 54.3 |
| Q8_0 | 0.53 | 102.9 | 54.5 |
| Q4_K_M | 0.39 | 136.7 | 53.3 |

Three files. Three speeds. One bandwidth, give or take a rounding error. All three sit just under the measured 57. Cut the bytes per token and the speed goes up by almost exactly the same factor. Whether the line holds for bigger models, and what the other cores are doing meanwhile, is [the next entry]({{ site.baseurl }}{% post_url 2026-09-20-your-cpu-is-asleep %}).

## Why

The measured part first. Adding cores barely moves the number, so the cores are not the limit. The limit is a door, and it lets bytes through at a fixed rate no matter how many cores queue behind it. We call it **the Bouncer**. We think the Bouncer is the memory controller. That is a guess for now; the circumstantial evidence comes later in the series.

Some of the gap between 96 and 57 is normal. No machine collects the whole spec-sheet number, because DRAM spends time refreshing itself and opening and closing rows. Not all of the gap is that, though. We have a theory about the rest. The BIOS knows, and [nobody has asked it]({{ site.baseurl }}{% post_url 2026-09-22-is-our-ram-at-half-speed %}).

The consequence is the plot of this whole series. The CPU can do far more arithmetic than the Bouncer can feed it. While the model writes, most of the cores are standing in the corridor. Most of the tricks in the posts that follow are ways to give them something useful to do.

You will meet the rest of the cast along the way:

- **The Target**: Qwen2.5-14B, the big model we actually want answers from. Every token it writes means a trip past the Bouncer with nearly every weight it owns.
- **The Intern**: Qwen2.5-0.5B, a small model that guesses the Target's next few words so the Target only has to check them. [Hiring it]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}) went better than it had any right to.
- **The Spinners**: idle worker threads that busy-wait instead of sleeping. They are not lazy. They are the opposite of lazy, and that is the problem.
- **The VIP Lounge**: the 96 MB cache on the one die that has 3D V-cache stacked on top. Someone is going to get a pass to it. It will not go the way we planned.

Also coming: in a synthetic test, reading only the rows you need pays off almost in full, if the idle cores help. Checking a few of the Intern's guesses costs about what checking one does. And every Q4_0 model we had crashed on our build, for a reason that has sat in the gcc bug tracker for a very long time.

## What you should do about it

- **Measure your own ceiling before you believe anyone's tokens per second, including ours.** A STREAM-style read test takes a minute. Judge decode speed against the measured number, not the spec sheet. Against 96, every dense model on this blog looks lazy. Against 57, they are doing their jobs.
- **Predict before you download.** For a dense model, your best case is your measured GB/s divided by the GB it reads per token. That is roughly the file size. Small models are the exception: a big slice of their file is a word-lookup table, and each token reads only one row of it. Our Q4_K_M 0.5B reads 0.39 GB per token. That predicts 146 tok/s. It delivered 136.7. If the prediction is too slow for you, no thread count will fix it. A smaller file will, and so will the Intern. Mixture-of-experts models read only part of their file per token, and they [get their own post]({{ site.baseurl }}{% post_url 2026-09-20-thirty-five-billion-three-active %}).
- **For writing speed, buy bandwidth, not cores.** Faster RAM and more memory channels raise the ceiling. Past a handful of cores, more cores mostly wait in the corridor, at least until the Intern shows up. Reading a long prompt is a different, arithmetic-heavy story, told later.

Next entry: [twelve models, one wall, and a 16-core CPU that barely notices when we send half its cores home]({{ site.baseurl }}{% post_url 2026-09-20-your-cpu-is-asleep %}).

## The receipt

Every post here ends the way this one does: with a receipt. It lists every number, where it came from, how many runs stand behind it and what is still pending. The joke never bends the number.

```receipt
GPU POOR LABS                       2026-09-20
------------------------------------------------
6000 MT/s x 8 B x 2 channels
  = spec-sheet bandwidth ... 96 GB/s  [F-P0-1]
Measured read peak ......... 57 GB/s  [F-P0-2]
  (range over thread configs 55.8-58)
One core alone ......... 52-54 GB/s   [F-P0-3]
Score: 57 / 96 ................ 59%   [derived]
Gap: 96 - 57 ............... 39 GB/s  [derived]
  part normal DRAM overhead, part not
0.5B F16   54.8 t/s x 0.99 GB = 54.3  [F-P1-2]
0.5B Q8_0 102.9 t/s x 0.53 GB = 54.5  [F-P1-2]
0.5B Q4KM 136.7 t/s x 0.39 GB = 53.3  [F-P1-2]
  (GB/token rounded; last digit is soft)
Predicted Q4KM: 57 / 0.39 .. 146 t/s  [derived]
Theory for the "part not" ..... GUESS [F-P10-4]
  (unconfirmed; the BIOS has not been asked)
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 16 cores, 2 dies
          96 MB L3 on the V-cache die
          2x48GB DDR5-6000 (XMP) = 96 GB
          RTX 5080 present, never used
BUILD     llama.cpp a894dae, gcc 14
          GGML_OPENMP=OFF, 16 threads
RUNS      STREAM: best rep per thread config;
          rerun queued (two records differ)
          tok/s: llama-bench, median of 3
THANK YOU FOR NOT BUYING A GPU
```
