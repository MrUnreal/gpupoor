---
layout: post
title: "We hired two mind-readers to replace the Intern. The Intern won."
date: 2026-09-22 14:50:00 -0400
sticker: "2.17× vs 1.89×"
sticker_note: "the Intern vs the best mind-reader"
tags: [speculative-decoding, eagle3, drafting]
lab: P12
summary: "Two public EAGLE-3 heads read our 14B Target's hidden states. We converted both to GGUF and raced them on a CPU. A plain 0.5B draft model still won, 2.17× to 1.89×."
---

The Intern is a 0.5B model that guesses the Target's next words by being a smaller version of the Target. He is cheap, he is fast, and he has no idea what the Target is thinking. He just has a similar upbringing.

EAGLE-3 heads are the upgrade GPU servers reach for. A head is a single transformer layer. It reads the Target's own hidden states, mid-thought, and guesses the next few tokens from them. It is a draft model that reads minds. We found two public heads for our Target, Qwen2.5-14B-Instruct. We hired both.

## Getting the mind-readers through the door

First, they could not get in. llama.cpp supports EAGLE-3 drafts. But its Qwen2 graph never saved the per-layer hidden states a head needs to read. The first request died with "layer input tensor is null".

The fix is one line in `src/models/qwen2.cpp`. The Qwen3 and Llama graphs already have it. Upstream made the same one-line change for another model family in [PR #25141](https://github.com/ggml-org/llama.cpp/pull/25141). Plain decoding is unaffected. By our reading of the code, the same line also unblocks DFlash and DSpark drafts against a Qwen2 Target. We only ran EAGLE-3.

Second, neither head came as a GGUF file. The stock converter handled both without complaint. It found the Target layers each head reads and wrote the files. We quantized both heads to Q8_0, and Head A also to Q4_K_M. As far as we can tell, these are the first GGUF builds of either head.

The two candidates:

- **Head A**, trained with SpecForge on UltraChat. Its draft vocabulary is 16k tokens.
- **Head B**, trained with SpecJAX on mixed chat data. Its draft vocabulary is 32k tokens.

Both authors published their heads in the open. That is the only reason this post exists.

## The interview

Target: Qwen2.5-14B-Instruct at Q4_0, the faster Target for writing in [the quant rematch]({{ site.baseurl }}{% post_url 2026-09-22-q4-0-vs-q4-k-m-rematch %}). Greedy decoding, 16 threads. Three prompts (code, prose, a structured list), two reps each. We measured the Target's plain speed before and after the whole session. It drifted 0.2%. For once, the machine behaved.

| drafter | K = 2 | K = 4 | K = 6 | K = 8 |
|---|---|---|---|---|
| The Intern (0.5B, Q4_K_M) | – | **2.05×** | – | **2.17×** |
| Head A, Q4_K_M | – | 1.89× | – | – |
| Head A, Q8_0 | 1.64× | 1.76× | 1.52× | 1.53× |
| Head A, bf16 | – | 1.49× | – | – |
| Head B, Q8_0 | 1.58× | 1.65× | 1.38× | – |

*Speedup over the Target decoding alone, mean of the three prompts. K is the number of guesses per round. A dash means we did not run it.*

The best mind-reader reached 1.89×. The Intern reached 2.17×. At the same K = 4, the Intern still won, 2.05× to 1.89×. On the code prompt at K = 8, the Intern hit 2.99×. The mind-readers never got near that.

## Why the Intern won

**Measured:** the best head is cheaper per round, but it commits fewer tokens per round. Head A at Q4_K_M took 183 ms per round. The Intern at K = 4 took 219 ms. But the Intern landed 3.16 tokens per round. The heads landed about 2.4. A cheap wrong guess is still a wrong guess.

**Measured:** at K = 4, the Target accepted 30–42% of the heads' guesses. It accepted 43–76% of the Intern's. The 76% was code. Longer drafts hurt the heads. Both peaked at K = 4. The Intern kept gaining at K = 8. Head A's per-token acceptance fell by more than half from K = 2 to K = 8, on every prompt. On code it went from 60% to 24%.

**Measured, then checked against the label:** we do not think this is a bug on our side. Head B's model card reports per-position acceptance of 60%, 57%, 55% and 54% on held-out chat data. Multiply those through for four guesses and you land close to what we got. We measured 2.36 tokens per round. Their Target was unquantized and their prompts were chat, so this shows consistency, not proof. Head A's card lists only GPU speedups, so it gets no such check.

**Measured, and familiar:** the head's precision did not change its guesses. At bf16 and at Q8_0, Head A had 41/34/36% of its guesses accepted on code/prose/list. Not roughly. The counts matched to the token. Q4_K_M landed within a point. Precision changed only the cost. bf16 gave 1.49×. Q4_K_M gave 1.89×. [The Intern taught us the same lesson]({{ site.baseurl }}{% post_url 2026-09-20-hire-the-cheapest-intern %}). His acceptance stayed at 51–55% whatever precision we gave him. Buy the cheapest quant of whatever drafts for you.

**Guessed** (not tested): three reasons the mind-readers underperform here.

- They learned to read the unquantized Target. Ours is squeezed to 4 bits, so its hidden states and its choices drift from what they studied. Same mind, lower resolution.
- They can only propose words from a 16k or 32k slice of the vocabulary.
- Tree drafting, for Head B only. Its card's GPU recipe drafts a tree of candidate branches and checks them at once. llama.cpp's server drafts a single chain, on any hardware (we read the code, not a benchmark). Head A's published speedups used a single chain, so this excuse does not cover Head A.

## Did anyone's answers get worse?

No. We scored every output under the Target, token by token, using the exact token ids the server generated. On the code and prose prompts, all eleven drafted configurations (both heads and the Intern) produced text byte-identical to the Target decoding alone.

On the list prompt, all eleven produced the same alternate text. The plain Target says neon is used in neon signs. Every drafted run says advertising signs. Both are true.

In greedy speculative decoding the Target picks every token; a drafter only changes how many get checked at once. So a detour shared by eleven drafters belongs to the checking, not to any of them. Our reading: batched checking is not bit-identical to one-token decoding, and here it flipped a near-tie. The detour is marginally more probable under the Target than the plain text.

## What you should do about it

- **Try a small sibling model before a learned head.** On this CPU, with a 4-bit Target, the plain 0.5B Qwen draft beat both public EAGLE-3 heads for Qwen2.5-14B.
- **If you use a head, quantize it.** Q4_K_M took Head A from 1.49× to 1.89×, and its acceptance barely moved.
- **Keep head drafts short.** K = 4 was best for both heads. K = 6 was worse than K = 2 for both.
- **Running EAGLE-3 against a Qwen2 Target in llama.cpp** needs the one-line `t_layer_inp` registration in `src/models/qwen2.cpp` until upstream has it. By our reading, DFlash and DSpark need it too.

## The receipt

```receipt
GPU POOR LABS                         2026-09-22
------------------------------------------------
TARGET  14B Q4_0, greedy, 16 threads
        3 prompts x 2 reps, one session
Plain drift, before/after ..... -0.2% [F-P12-11]
------------------------------------------------
Intern, K=4 ................... 2.05x [F-P12-3]
Intern, K=8 ................... 2.17x [F-P12-3]
Intern, code, K=8 ............. 2.99x [F-P12-3]
Head A Q4_K_M, K=4 (best) ..... 1.89x [F-P12-4]
Head A Q8_0, K=2 / 4 ..... 1.64/1.76x [F-P12-9]
Head A Q8_0, K=6 / 8 ..... 1.52/1.53x [F-P12-9]
Head A bf16, K=4 .............. 1.49x [F-P12-6]
Head B Q8_0, K=4 (best) ....... 1.65x [F-P12-4]
Head B Q8_0, K=2 / 6 ..... 1.58/1.38x [F-P12-9]
------------------------------------------------
ms/round, K=4: Head A Q4_K_M .... 183 [F-P12-10]
ms/round, K=4: Intern ........... 219 [F-P12-10]
Tok/round, K=4: Intern ......... 3.16 [F-P12-5]
Tok/round, K=4: heads ..... 2.36-2.44 [F-P12-5]
Tok/round, K=4: Head B ......... 2.36 [F-P12-5]
Accepted, K=4: heads ......... 30-42% [F-P12-5]
Accepted, K=4: Intern ........ 43-76% [F-P12-5]
Head A accepted, code, K=2 ...... 60% [F-P12-12]
Head A accepted, code, K=8 ...... 24% [F-P12-12]
Head B card, pos 1-4 ... 60/57/55/54% [F-P12-7]
  (their data: held-out chat)
Head A accepted, code/prose/list, K=4
  bf16 = Q8_0, exactly .... 41/34/36% [F-P12-6]
Intern accepted, 3 quants .... 51-55% [F-P3-4]
------------------------------------------------
Code + prose = plain greedy ... 11/11 [F-Q-3]
List: same alternate text ..... 11/11 [F-Q-3]
------------------------------------------------
HARDWARE  Ryzen 9 9950X3D, 2x48GB DDR5-6000
BUILD     llama.cpp a894dae + P6/P7 patches
          + qwen2 t_layer_inp (1 line)
HEADS     2 public, converted here to GGUF
UNTESTED  bf16 Target, tree drafting,
          DFlash/DSpark on Qwen2
THANK YOU FOR NOT BUYING A GPU
```
