---
layout: post
title: "OpenMP made our CPU 10× slower and nobody said a word"
date: 2026-09-20 13:00:00 -0400
sticker: "10×"
sticker_note: "slower, for free, with OpenMP"
tags: [build, windows, threads, openmp]
lab: ADR-0002
summary: "Our first llama.cpp build on Windows got slower every time we added threads, from 70 tok/s on one to 13 on sixteen; one CMake flag later, 8 to 16 threads gave 126–133."
---

The first llama.cpp benchmark on the first day was supposed to be a formality. Take the smallest model we had downloaded, Qwen2.5-0.5B, give it all 16 cores, watch the plumbing work. It decoded at 13 tokens per second. Then we gave it fewer cores, and it got faster.

## What we measured

### Two detours before the first number

Clang 19 was the first choice. It wanted `-D_WIN32_WINNT=0x0A00` before the bundled HTTP library would compile. Then it failed at link time, asking for a libstdc++ symbol called `__once_callable` that it was never going to get. gcc 14 from the same WinLibs bundle linked cleanly. We did not argue. (gcc and the number 32 have [their own falling-out]({{ site.baseurl }}{% post_url 2026-09-21-gcc-cannot-count-to-32 %}) later in this diary.)

The build succeeded. The binaries did not. They exited without printing a thing. Git for Windows puts its own, older mingw runtime earlier on `PATH`, and Windows loaded those DLLs instead of the ones our binaries were built against. The fix is dull and permanent: copy `libstdc++-6.dll`, `libgcc_s_seh-1.dll`, `libwinpthread-1.dll` and `libgomp-1.dll` from the toolchain into the folder with the executables. Windows looks there first.

### The sweep

We left OpenMP at its CMake default, which is on. CMake found gcc's OpenMP runtime, libgomp, mentioned it in a few quiet lines of configure output, and wired it in. Then we ran the cheapest benchmark there is: one model, decode only, more and more threads. Then we rebuilt with `-DGGML_OPENMP=OFF`, which swaps libgomp for ggml's own thread pool, and ran it again.

| Build | Threads | Qwen2.5-0.5B (Q4_K_M) decode, tok/s |
|---|---:|---:|
| gcc + libgomp | 1 | 70 |
| gcc + libgomp | 4 | 36 |
| gcc + libgomp | 8 | 22 |
| gcc + libgomp | 16 | 13 |
| ggml's own pool | 8–16 | 126–133 |
| ggml's own pool | 32 (SMT) | 13 |

*Same compiler, same llama.cpp commit, same model. The two builds differ by one CMake flag.*

Read the libgomp rows top to bottom. Every step up in threads made decode slower. Sixteen threads were 5.4× slower than one.

Now the fifth row. That is ggml's own pool, at 8 to 16 threads. It runs roughly 10× faster than the OpenMP build at 16. It cost one CMake flag. Nothing warned us. llama-bench printed 13 tok/s with the same calm it prints everything, and fairly so: it has no way to know what the number should have been.

Then, to be thorough, we gave the good build all 32 hardware threads. It fell to 13 tok/s. That is the broken build's number, reached by a completely different road. In this lab, 13 is what the machine says when you are holding it wrong.

## Why

**From the code.** Decoding a token runs a graph of small operations: a matrix-vector product, a norm, a rotation, another product. ggml, the engine inside llama.cpp, puts a barrier after every one. A barrier is a meeting nobody may leave until everyone has arrived. One token is a lot of meetings.

**Measured, then inferred.** The work per token does not grow with the thread count. It is the same weights, read once, and memory sets the pace; [half the cores already get almost all of it]({{ site.baseurl }}{% post_url 2026-09-20-your-cpu-is-asleep %}). So when speed falls as threads rise, the extra time is not work. We read it as coordination, and it grows with the headcount.

**Guessed.** libgomp's barrier on Windows costs more per meeting than ggml's, and more again with every attendee. We tried OpenMP's own settings for how waiting threads should behave, spin or sleep. Nothing moved. We have not profiled libgomp, and we only measured Windows with mingw; Linux may be perfectly happy. If you know exactly where the time goes, we would love to hear it.

**From the code, again.** ggml's own pool meets differently. The barrier is an atomic counter, and threads that arrive early spin on it instead of asking Windows to wake them later. These are the Spinners. Spinning is cheap as long as every Spinner has a core to itself. At 16 threads on 16 cores, each one can.

**Guessed, for SMT.** At 32 threads, every core carries two. A Spinner can now share a core with the thread doing the real work, and burn the cycles that thread needs. And if Windows pauses any one thread for a moment, everyone else stands at the barrier, waiting for a colleague who has left the building.

Remember the Spinners. By default, speculative decoding keeps the Target's 16 idle threads spinning while the Intern works with 16 threads of its own, on the same 16 cores. That is 32 threads again, [wearing a nicer suit]({{ site.baseurl }}{% post_url 2026-09-21-we-paid-sixteen-threads-to-spin %}).

## What you should do about it

- **Building llama.cpp for CPU on Windows with gcc: pass `-DGGML_OPENMP=OFF`.** Then run `llama-bench -t 1,4,8,16`. If 16 threads is slower than 1, you are paying the tax.
- **Threads = physical cores.** On this chip llama.cpp's default already lands on 16 (a mingw build takes half the logical CPUs), which is exactly right. The trap is "helpfully" passing `-t 32`.
- **If a fresh binary exits silently, suspect someone else's DLLs.** `where.exe libstdc++-6.dll` lists every copy on `PATH`, in search order. Copy your toolchain's runtime DLLs next to the executables and move on.
- **Clang 19 dying on `__once_callable` with WinLibs:** use the gcc 14 from the same bundle. We kept `-D_WIN32_WINNT=0x0A00` in the gcc build's C and C++ flags too.

## The receipt

```receipt
GPU POOR LABS                         2026-09-20
------------------------------------------------
Qwen2.5-0.5B Q4_K_M decode, tok/s
libgomp,  1 thread ............ 70   [F-B-1]
libgomp,  4 threads ........... 36   [F-B-1]
libgomp,  8 threads ........... 22   [F-B-1]
libgomp, 16 threads ........... 13   [F-B-1]
ggml pool, 8-16 threads ... 126-133  [F-B-1]
ggml pool, 32 threads (SMT) ... 13   [F-B-2]
Spec decode default ..... 16+16 thr  [F-P6-1]
------------------------------------------------
libgomp, 1 vs 16 threads
  70 / 13 ................ 5.4x derived
ggml pool (8-16 thr) / libgomp (16)
  126-133 / 13 ...... 9.7-10.2x derived
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
          16 cores, 32 threads, Windows 11
BUILD     llama.cpp a894dae, gcc 14.2 WinLibs
          GGML_OPENMP=ON, then OFF
          clang 19: link failed, never ran
RUNS      1 sweep per build, first day
TRIED     OpenMP wait settings: no effect
GUESSED   why libgomp's barrier is slow
          (not profiled); why SMT hurts
NOT RUN   Linux, libgomp at 32 threads
THANK YOU FOR NOT BUYING A GPU
```
