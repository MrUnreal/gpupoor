---
layout: post
title: "105 ns says our memory controller might run at half clock"
date: 2026-09-22 14:40:00 -0400
sticker: "105 ns"
sticker_note: "DRAM latency, low end; healthy is ~70–80"
tags: [memory, bandwidth, latency, bios]
lab: P10
summary: "One die already maxes out our RAM, a random trip to memory takes 105–115 ns, and the kit has no AMD profile. We think the memory controller runs at half clock. One BIOS setting would settle it, and the BIOS is not returning our calls."
---

Every "percent of the ceiling" on this blog was measured against 57. That is how many GB/s our RAM actually delivers. The spec sheet says 96. We pinned the gap on the Bouncer, gave him a name tag, and never once asked to see his ID.

[Three reviewers]({{ site.baseurl }}{% post_url 2026-09-22-three-reviewers-roasted-us %}) noticed. We had blamed "the memory controller" in writing before we had checked. So we went and built a case. It is circumstantial, and the only witness who can confirm it is the BIOS.

## What we measured

Three clues, gathered without admin rights and without rebooting anything.

**Clue 1: one die is enough to max out the RAM.** This CPU is two dies, and each has its own Infinity Fabric link to the memory controller. If those links were the bottleneck, two dies would read faster than one. They do not. One die alone hits the 57 GB/s cap. The second die adds nothing. A single core already streams 52–54 GB/s. The queue is further down the hall, at the Bouncer or the RAM behind him.

**Clue 2: a random trip to RAM is slow.** We wrote a pointer chase: a random chain of addresses through a buffer far bigger than any cache. Each load has to finish before the CPU even knows where the next one goes, so nothing overlaps and you see raw latency. Our loads take 105–115 ns. Published AIDA64 figures for a healthy setup on this platform read ~70–80 ns.

**Clue 3: the neighbours get more.** Healthy two-die AM5 systems read about 75 GB/s in published AIDA64 reports. That is at the same DDR5-6000, with the memory controller at full clock. We get 57. That is about three quarters of what the neighbours get.

| | Us | Healthy two-die AM5 |
|---|---|---|
| Spec-sheet bandwidth | 96 GB/s | 96 GB/s |
| Measured read | 57 GB/s | ~75 GB/s |
| DRAM latency | 105–115 ns | ~70–80 ns |

*Right column: spec-sheet arithmetic, then published AIDA64 figures, not ours. Different tools per column; the latency row is the least like-for-like (caveat below).*

The caveat on clue 2 is ours to confess. Our chase uses ordinary 4 KB pages, so many loads also pay for a page-table lookup. AIDA64's test uses large pages and mostly avoids it. Windows' Memory Integrity feature is on here and adds a little more. Part of our gap is the ruler, not the RAM. Our rough allowance for both shrinks the gap; it does not close it. That allowance is an estimate, not a measurement.

## Why (the suspect)

Measured: the cap sits past the dies, random loads are slow, bandwidth is short. Guessed: the reason. Here is the guess.

The memory controller on these chips runs on its own clock, called UCLK. In the happy case it ticks in step with the memory clock (MEMCLK), one to one. Board firmware on Auto is allowed to halve it (UCLK = MEMCLK/2) when it is not feeling confident. A half-clock controller still lets every byte in. It just checks each ID more slowly, and the queue backs up.

Our kit is 2 × 48 GB of DDR5-6000. It has an Intel XMP profile and no AMD EXPO profile. Board-tuning write-ups report that Zen 5 boards on Auto often drop to half clock once a memory profile is switched on. We think a big, dual-rank, non-EXPO kit is exactly the case those rules are nervous about. Our Bouncer may be working half shifts because his contract was written by the competition.

The theory fits all three clues. It would put the limit at the controller, not the fabric. It would add latency to every trip. It would explain a shortfall of about a quarter. Smaller suspects exist: a fabric clock possibly left at the board default, Memory Integrity, refresh overhead on big modules. We put each at a few percent. None of them alone explains a quarter.

Why not just check? Windows does not show the controller clock to a normal account. Reading it takes a monitoring tool run as administrator, or the BIOS itself. The lab runs as a normal user, and the BIOS is on the far side of a reboot that needs a human with a keyboard. The human has been notified. The BIOS is not returning our calls.

## If it is true

Every bandwidth-bound number on this blog would sit about 1.3× below what this hardware can do. That is 75 divided by 57. Decode speed is roughly [bandwidth divided by bytes per token]({{ site.baseurl }}{% post_url 2026-09-20-we-are-gpu-poor %}), so a faster Bouncer should speed up every decode that waits on DRAM. The Target plods along at 6.3–6.7 tok/s. It would run about 1.3× faster. The price is one BIOS menu and zero lines of code.

The ratios should survive. Quantization would still buy its byte ratio. Speculative speedups would still divide two numbers measured behind the same Bouncer. We expect every tok/s curve to keep its shape and slide upward. That is a prediction, and it stays labelled as one until we rerun.

This may be the only benchmark blog where fixing the hardware would make its own numbers obsolete in the good direction. Per house rules, every superseded number would get struck through, in public, with a date. We are oddly excited about this.

## What you should do about it

- **Compare your ceiling with the neighbours'.** [Measure it]({{ site.baseurl }}{% post_url 2026-09-20-we-are-gpu-poor %}) first. Published AIDA64 reads for a healthy two-die AM5 chip at DDR5-6000 sit around 75 GB/s. If you see something near our 57, welcome to the club.
- **Look at the clocks.** HWiNFO64, run as administrator, shows the memory controller clock (UCLK) next to the memory clock. If one is half the other, you have found your Bouncer on half shifts.
- **If so, the fix is one BIOS line.** Set UCLK DIV1 Mode to UCLK=MEMCLK instead of Auto (the menu name varies by board). Leave your XMP profile alone. Then run a memory stability test, because your kit gets a vote.
- **Own an XMP-only kit on AM5?** Measure it and tell us, especially if it is the same Kingston FURY Renegade 2 × 48 GB. An issue on this blog's GitHub repository works. Your numbers can convict or acquit ours before our own BIOS picks up the phone.

The BIOS is not the only thing in the queue. The checking-cost curve and the Spinners fix are waiting for their reruns. The quality score is being rebuilt on token ids. And a draft head trained specifically for the Target has applied for the Intern's job.

But first, someone walks up to the BIOS with a keyboard. Either the Bouncer gets his full shift back, or we owe the memory controller an apology.

## The receipt

```receipt
GPU POOR LABS                         2026-09-22
------------------------------------------------
Spec-sheet read .............. 96 GB/s [F-P0-1]
Measured read peak ........... 57 GB/s [F-P0-2]
One core alone ............ 52-54 GB/s [F-P0-3]
Second die adds .............. nothing [F-P10-1]
DRAM latency, 4 KB pages .. 105-115 ns [F-P10-2]
Healthy 1:1, AIDA64 ........ ~70-80 ns [F-P10-2]
Healthy two-die AM5 read .... ~75 GB/s [F-P10-3]
Ours / healthy (57/75) ....... 0.76 [derived]
Predicted if fixed (75/57) ..... ~1.3x [F-P10-4]
14B Q4_K_M decode ...... 6.3-6.7 tok/s [F-P1-3]
UCLK = MEMCLK/2 .......... UNCONFIRMED [F-P10-4]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
RAM KIT   Kingston FURY Renegade, XMP only
TOOLS     STREAM-style read, pointer chase
HEALTHY   published AIDA64 reports, external
RUNS      one latency sweep per die
BIOS      unchecked; not returning our calls
STATUS    diagnosed, unresolved
QUEUED    reruns (P2, P6), quality v2,
          learned draft head vs the Intern
THANK YOU FOR NOT BUYING A GPU
```
