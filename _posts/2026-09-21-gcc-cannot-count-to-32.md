---
layout: post
title: "gcc can't count to 32 on Windows, and hasn't since 2012"
date: 2026-09-21 14:00:00 -0400
sticker: "16 ≠ 32"
sticker_note: "bytes of stack alignment, promised vs needed"
tags: [gcc, windows, llama-cpp, q4_0, debugging]
lab: P7
summary: "Every Q4_0 model crashed on our Windows build because gcc wrote a 32-byte vector into a stack slot aligned for 16. No gcc flag we tried fixes it. A pointer does."
---

Every Q4_0 model we tried crashed right after loading. It didn't give a wrong answer or run slowly. It segfaulted in every llama.cpp tool we pointed at it, on one thread or sixteen. Q4_K_M, Q8_0 and F16 all ran fine, and only the most boring quant in the catalogue refused to exist.

## What we measured

We were on gcc in the first place because clang would not link, a saga told in [the OpenMP post]({{ site.baseurl }}{% post_url 2026-09-20-openmp-tax %}). This was the toolchain's second ambush. We attached gdb and let it crash. It crashed, obligingly, on one instruction (gdb output, trimmed):

```text
Thread 7 received signal SIGSEGV, Segmentation fault.
#0  ggml_gemv_q4_0_8x8_q8_0 ()
=> <ggml_gemv_q4_0_8x8_q8_0+57>:  vmovdqa %ymm0,0x40(%rsp)
rsp            0x314f8b0
```

This `vmovdqa` writes 32 bytes from a ymm register. The "a" stands for *aligned*, and the address must be a multiple of 32. That is a requirement, not a suggestion.

The target address is `0x314f8b0 + 0x40 = 0x314f8f0`. In hex, a multiple of 32 ends in an even digit and a zero. This one ends in `f0`. The slot was aligned to 16 bytes. The instruction needed 32.

The function is a thin wrapper. It builds a lookup table that turns 4-bit weights into signed bytes. Then it passes the table, as one 32-byte vector, *by value* to the real kernel. On the Windows x64 ABI, a vector that big travels as a pointer to a temporary copy on the caller's stack. gcc put that copy at a 16-byte-aligned address. Then it wrote the copy with an instruction that demands 32.

Then we tried to fix it, and nothing worked:

- **The flag built for this, `-mstackrealign`.** We rebuilt. It crashed on the same instruction. The flag turns out to be on by default for this target. We had asked gcc to try harder at the thing it was already failing at.
- **Passing the table by reference.** We rebuilt, and the wrapper's machine code came out byte-for-byte identical.
- **Adding `alignas(64)` on top of the reference.** Byte-identical again. Our best guess is that gcc's interprocedural optimizer cloned the kernel (we saw an `.isra.0` clone) and turned the reference back into a by-value copy. In other words, the compiler optimized our fix away. We did not test with that pass disabled.

Between rebuilds we also wrote a napkin-sized reproducer, away from llama.cpp: one `__m256i` local whose address escapes. Then we tried every gcc flag that sounded helpful:

| flags | what gcc emitted |
|---|---|
| `-O3 -mavx2` or `-march=native` | 16-byte-aligned frame, 32-byte aligned store |
| `-mstackrealign` | no change |
| `-mincoming-stack-boundary=3` | realigns the stack… to 16 |
| `-fno-omit-frame-pointer` | same store via `%rbp`, still 16-aligned |
| `-mpreferred-stack-boundary=5` | refused: "not between 3 and 4" |

The last row is where the title comes from. The option takes an exponent of two: 3 means 8 bytes, 4 means 16 and 5 means 32. We asked for 32. gcc said the answer must be between 8 and 16.

## Why

Measured, on the reproducer: this gcc, targeting Windows, keeps its stack frames aligned to 16 bytes at most. It still emits 32-byte aligned stores into them. Whether a given function faults depends on where its caller left the stack pointer. That makes any function that keeps a plain `__m256i` on its stack a coin flip. From the disassembly, the K-quant kernels survive because they happen not to put a ymm register on the stack, not because they are better behaved.

The why comes from reading gcc's source, not from measurement. On Windows, gcc unwinds the stack with SEH, structured exception handling. SEH unwind info cannot describe a frame that was realigned at run time. So on SEH targets gcc caps stack alignment at 16 bytes. It does not realign for an ordinary 32-byte vector. It uses the aligned store anyway. Two parts of the compiler made two different promises, and our wrapper was standing between them.

This is [gcc bug 54412](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=54412). It has been open since 2012.

It is not a careless bug. It lives where an ABI, an exception unwinder and a vector extension meet, and nobody's weekend covers all three. Newer w64devkit toolchains ship a mitigation: they assemble with `-muse-unaligned-vector-move`, which quietly swaps aligned vector moves for unaligned ones. Our WinLibs gcc 14 does not do that by default.

The crash had been spotted before. [llama.cpp #16479](https://github.com/ggml-org/llama.cpp/issues/16479) reported it on an older toolchain. The maintainers reproduced it and noted that the crashing function did not contain a single pointer, which is the most haunted sentence you can write about a segfault. They correctly blamed the compiler and suggested a newer w64devkit, and the issue closed as stale. Fair call, since newer w64devkit builds carry that assembler flag. A newer gcc alone does not help. Ours is gcc 14, and it crashed anyway.

**The fix** is to never put a 32-byte vector on the stack at the call. The kernel now takes a pointer to the 16-byte source table and builds the 32-byte vector in a register. The wrapper shrinks to "load an address, jump". The patch adds 18 lines and removes 20. The PR text is written and not yet filed. It removes the one guaranteed instance, and gcc can still spill a ymm register somewhere else under pressure. Only a toolchain fix covers that.

**The payoff:** Q4_0 finally ran. Qwen2.5-7B at Q4_0 prefills at 217–243 tok/s. The Q4_K_M we had been using does 155 on stock llama.cpp. Decode runs at 13.5–14.4 tok/s. The most boring quant in the catalogue out-prefilled the fancy one. It also had the flattest verification cost at 16 tokens of the quants we tried. That is one run per point, with a rerun queued. The blue Q4_0 line in [the verification post]({{ site.baseurl }}{% post_url 2026-09-20-checking-four-answers %}) only exists because of this fix. Enjoy the upset while it lasts, because [the rematch]({{ site.baseurl }}{% post_url 2026-09-22-q4-0-vs-q4-k-m-rematch %}) has a PR in it that flips prefill.

## What you should do about it

- **Q4_0 crashes on your Windows gcc build?** Run with `--no-repack`. It runs. You give up the repacked kernels, and the prefill numbers above came from those.
- **Fix the toolchain instead of the flags.** Use a w64devkit that ships the mitigation, or add `-Wa,-muse-unaligned-vector-move` to a gcc + MinGW build of ggml. Or use clang or MSVC, if they link for you. We have tested none of these three ourselves. Skip `-mstackrealign` and `-mpreferred-stack-boundary=5`: we tried both; one changes nothing and the other is refused.
- **Writing SIMD for mingw gcc?** Never pass an `__m256i` by value to a function that might not inline, and do not trust `alignas` to survive the optimizer. Pass a pointer to the bytes and build the vector in a register.
- **Audit a binary** with `objdump -d <binary> | grep -E 'vmov(dqa(32|64)?|aps|apd) +%[yz]mm[0-9]+,.*\(%r[sb]p\)'`. It is a rough filter we have not run ourselves. Each hit is a suspect, not a conviction.

## The receipt

```receipt
GPU POOR LABS                         2026-09-21
------------------------------------------------
Q4_0 models that ran, repack on ..... 0 [F-P7-1]
Faulting instr: vmovdqa %ymm0,0x40(%rsp)
  alignment it needs ............. 32 B [F-P7-1]
  alignment of the slot .......... 16 B [F-P7-1]
  0x314f8f0 mod 32 (derived) ....... 16 [F-P7-1]
SEH stack alignment cap .......... 16 B [F-P7-2]
-mpreferred-stack-boundary=5 .. refused [F-P7-2]
gcc bug 54412 open since ......... 2012 [F-P7-2]
Patch lines added/removed ..... +18/-20 [F-P7-3]
PR filed ............... no, text ready [F-P7-3]
7B Q4_0 prefill, fixed .... 217-243 t/s [F-P7-4]
7B Q4_0 decode, fixed ... 13.5-14.4 t/s [F-P7-4]
7B Q4_K_M prefill, stock ...... 155 t/s [F-P1-6]
Q4_0 verify cost, 16 tokens .. flattest [F-P2-2]
  1 run per point, 3-rep rerun queued
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
TOOLCHAIN gcc 14 (WinLibs, mingw-w64, SEH)
BUILD     llama.cpp a894dae (+ P7 fix, others)
RUNS      llama-bench; ranges span 2 sessions
CAVEAT    gcc may still spill ymm elsewhere
GUESS     IPA-SRA undid the alignas (untested)
THANK YOU FOR NOT BUYING A GPU
```
