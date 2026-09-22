# SERIES.md — the running plan

Status legend: `planned` → `drafted` → `fact-checked` → `published`. Posts 1–13: published 2026-09-22 (written by one agent each, adversarially fact-checked, series-edited). Posts 5 and 10 carry a pending pinning correction (F-PIN-1). Lab source paths are relative to the lab notebook root (`F:\Projects\Python\cpubound`). Slugs are final: other posts link to them with `{{ site.baseurl }}{% post_url SLUG %}`.

## Backlog batch (findings from 2026-09-20 to 2026-09-22)

### 1. `2026-09-20-we-are-gpu-poor` — published date 2026-09-20 09:00
- **Title idea:** "We are GPU poor, and our RAM is doing 57 out of 96"
- **Sticker:** `57 GB/s` — "what the RAM actually delivers"
- **Angle:** series opener and manifesto. The premise (desktop CPU, no GPU on purpose), the one equation (tokens/sec ≈ bandwidth ÷ bytes per token), the first surprise (spec sheet 96 GB/s, measured 57; one core already gets 54). Introduce the cast. Promise receipts. Tease what is coming (sparsity is cheap, verification is cheap, an Intern, a compiler bug).
- **Facts:** F-P0-1, F-P0-2, F-P0-3, F-P1-2 (as the proof the equation works), F-P10-4 only as a one-line tease ("we have a theory about the missing 39 GB/s; the BIOS knows").
- **Lab sources:** `docs/hardware.md`, `results/stream.json`, `docs/brief.md` §1, `docs/REPORT.md` §1–3.
- **Chart:** none (or a tiny table).

### 2. `2026-09-20-your-cpu-is-asleep` — 2026-09-20 11:00
- **Title idea:** "Your 16-core CPU is 14 cores asleep while it writes"
- **Sticker:** `98%` — "of full speed, using half the cores"
- **Angle:** bytes-per-token law holds on 12 models at 90–99% of the measured ceiling; quantization buys exactly its byte ratio; 8 cores = 98% of 16; one core is limited by the dot-product kernel, not RAM. Prefill is different (kernel-bound, ~40% of FP32 peak; the correction story is post 12's job, just state the right number). The idle cores are the resource the rest of the series spends.
- **Facts:** F-P1-1, F-P1-2, F-P1-3, F-P1-4, F-P1-5, F-P1-6, F-P0-2.
- **Lab:** `poc/P1-bytes-per-token/README.md`, `results/p1-baseline.jsonl`.
- **Chart:** `p1-bandwidth-line.png`.

### 3. `2026-09-20-openmp-tax` — 2026-09-20 13:00
- **Title idea:** "OpenMP made our CPU ten times slower and nobody said a word"
- **Sticker:** `10×` — "slower, for free, with OpenMP"
- **Angle:** the build-trap post. gcc+libgomp on Windows: more threads, slower decode (70 → 36 → 22 → 13 tok/s), ggml's own pool 126–133. SMT (32 threads) collapses the spin pool the same way. The mingw runtime DLL shadowing (Git for Windows' older libstdc++ earlier on PATH: the binaries silently failed to start). Clang 19 link failure → gcc. Practical checklist for building llama.cpp on Windows for CPU.
- **Facts:** F-B-1, F-B-2.
- **Lab:** `docs/adr/0002-llama-cpp-as-inference-substrate.md`, `docs/hardware.md` (toolchain rows), `PROGRESS.md` log entries for 2026-09-20 about the build.
- **Chart:** none; a small before/after table.

### 4. `2026-09-20-sparse-is-free-with-friends` — 2026-09-20 15:00
- **Title idea:** "Skipping 99% of the work is free, if you bring 15 friends"
- **Sticker:** `72×` — "speedup reading 1% of rows, ideal 100×"
- **Angle:** the textbook warning (sparse gathers trade bandwidth for latency) tested directly. True for one core, false for sixteen: memory-level parallelism from idle cores hides the latency. Software prefetch: no effect. 256-byte rows still pay. Implication: neuron-sparsity methods (Deja Vu, PowerInfer) stalled because nobody ships ReLU models, not because CPUs cannot gather. Careful: this is a synthetic kernel (no dequant math); say so.
- **Facts:** F-P4-1 … F-P4-5.
- **Lab:** `poc/P4-gather-vs-stream/README.md`, `bench/gather/gather.c`, `docs/survey/papers.md` §B for context.
- **Chart:** `p4-gather-vs-stream.png`.

### 5. `2026-09-20-checking-four-answers` — 2026-09-20 17:00
- **Title idea:** "Checking four answers costs the same as checking one"
- **Sticker:** `1.08×` — "cost of verifying 4 tokens vs 1"
- **Angle:** why speculative decoding should work on a CPU: verification of several tokens rides the same weight read. The measured curve (4 → 1.07–1.14, 8 → 1.24–1.35, 16 → 1.48–1.74), knee at 8–16, 8 cores steeper. A published Metal measurement suggested near-linear cost; x86 does not behave that way. Caveats on the receipt: 1 run per point, rerun queued, batched-bench is a proxy for verification.
- **Facts:** F-P2-1, F-P2-2, F-P2-3.
- **Lab:** `poc/P2-batch-scaling/README.md`, `docs/survey/gap-speculative-cpu.md` (for the Metal reference).
- **Chart:** `p2-verification-cost.png`.

### 6. `2026-09-20-hire-the-cheapest-intern` — 2026-09-20 20:00
- **Title idea:** "Hire the cheapest intern: 2× faster text from a 0.5B sidekick"
- **Sticker:** `2.0×` — "faster, same answers"
- **Angle:** speculative decoding explained with the Target and the Intern. The grid: six Interns (0.5B/1.5B/3B at several precisions) × draft lengths. The smallest Intern wins; bigger ones agree more but cost more; the Intern's precision does not matter; draft four tokens, never sixteen. Code 2.5×, prose 1.5× (the Intern is bad at prose). N-gram lookup does nothing on fresh text. Bonus: the MoE brought its own intern (MTP head) for free: 1.31×. Quality: outputs are not byte-identical but neither are 8- vs 16-thread plain runs (F-Q-1); do not claim more than that.
- **Facts:** F-P3-1 … F-P3-7, F-Q-1.
- **Lab:** `poc/P3-speculative-cpu/README.md`, `results/p3s-*.jsonl`, `docs/adr/0008-speculative-quality-metric.md`.
- **Charts:** `p3-draft-size.png`, `p3-draft-length.png`.

### 7. `2026-09-20-thirty-five-billion-three-active` — 2026-09-20 22:00
- **Title idea:** "35 billion parameters, 3 billion awake, 28% missing"
- **Sticker:** `28%` — "slower than it should be (we think we know why)"
- **Angle:** the MoE (Qwen3.6-35B-A3B) decodes at 19.6 tok/s where a dense 3B with the same weight bytes does 27.5. Not the expert gather (it is flat in thread count). First explanation (dispatch overhead) was wrong: fewer threads should have helped and did not. The reviewers' better guess: 30 of 41 layers carry a 2 MiB recurrent state read and written every token that nobody counted. Labelled hypothesis; per-op profile pending. Good "we were wrong in real time" material, lightly.
- **Facts:** F-P5-1, F-P5-2, F-P5-3, F-P3-7 (its MTP head, one line).
- **Lab:** `poc/P5-moe-vs-dense/README.md` (including the correction section).
- **Chart:** `p5-moe-threads.png`.

### 8. `2026-09-21-we-paid-sixteen-threads-to-spin` — 2026-09-21 10:00
- **Title idea:** "We paid 16 threads to spin in place. They ate 44% of our speedup."
- **Sticker:** `1.58× → 2.03×` — "same models, fewer Spinners"
- **Angle:** the Spinners. llama.cpp gives the Target a 16-thread pool that busy-waits after every step; the Intern had no pool at all (new OS threads every token) and its CPU flags were parsed and ignored. Two flags fix it; a ~60-line patch makes it the default. Tell it as a whodunit. Caveats: the A/B was not interleaved, rerun queued; the patch only matters with `GGML_OPENMP=OFF`; PR text written, not filed.
- **Facts:** F-P6-1, F-P6-2, F-P6-3.
- **Lab:** `poc/P6-spec-threadpool-pause/README.md`, `PR.md`, `pause-pools.patch`, `docs/survey/stack-map.md` §3.
- **Chart:** `p6-thread-layout.png`.

### 9. `2026-09-21-gcc-cannot-count-to-32` — 2026-09-21 14:00
- **Title idea:** "gcc can't count to 32 (on Windows, since 2012)"
- **Sticker:** `16 ≠ 32` — "bytes of stack alignment, promised vs needed"
- **Angle:** every Q4_0 model crashed. gdb, one instruction (`vmovdqa` to a 16-byte-aligned slot), two failed fixes (`-mstackrealign`, pass-by-reference, `alignas(64)`), the real reason (SEH targets cap stack alignment at 16 bytes; gcc bug 54412, open since 2012), a ten-line reproducer, the fix (pass a pointer). Be fair: newer w64devkit toolchains ship a mitigation (`-muse-unaligned-vector-move`); this toolchain does not by default. The crash had been reported before (#16479) and closed as an old-toolchain issue. Payoff: Q4_0 then turned out to be the fastest quant (prefill 217–243 tok/s).
- **Facts:** F-P7-1 … F-P7-4.
- **Lab:** `poc/P7-q4_0-gemv-msabi-fix/README.md`, `PR.md`, `bench/gcc-align/t.c`.
- **Chart:** none; a short disassembly snippet in a code block.

### 10. `2026-09-21-the-intern-got-a-vip-lounge` — 2026-09-21 18:00
- **Title idea:** "We gave the Intern the VIP lounge. Nothing happened."
- **Sticker:** `61.9 vs 62.2` — "tok/s, with and without the VIP lounge"
- **Angle:** the 96 MB V-cache. A 145 MB Intern runs 1.65× faster alone when it sits on the cache die (495 vs 301 tok/s, more than DRAM can deliver). Inside speculative decoding: no difference. Leading explanation: the Intern's in-loop cost is fixed per-token work, not bytes. Caveats (no positive pinning control). Rules that survived: give the Target every core, the Intern needs four threads, and a too-big Intern is a net loss.
- **Facts:** F-P8-1 … F-P8-4.
- **Lab:** `poc/P8-cache-resident-draft/README.md`.
- **Charts:** `p8-residency.png`, `p8-spec-layouts.png`.

### 11. `2026-09-22-q4-0-vs-q4-k-m-rematch` — 2026-09-22 10:00
- **Title idea:** "Q4_0 vs Q4_K_M: the rematch nobody asked for (3.0× on code)"
- **Sticker:** `3.0×` — "best run of the study"
- **Angle:** the quant fight. Round 1 (P9): Q4_0 is faster as a speculative target by its byte ratio; same relative gain; best run 21.3 tok/s from 7.0 on code at K = 8. Round 2 (P11): an open llama.cpp PR (#27851, tiled K-quant GEMM) flips prefill: Q4_K_M goes 162 → 204, or ~405 with `--no-repack` (the flag nobody would guess), Q6_K 110 → 391, Q4_0 unchanged, decode unchanged. Verdict: Q4_0 for generation-heavy and speculative work; Q4_K_M + the PR + `--no-repack` for long prompts.
- **Facts:** F-P9-1, F-P9-2, F-P11-1 … F-P11-4, F-P7-4.
- **Lab:** `poc/P9-q4_0-target/README.md`, `poc/P11-tiled-kquant-gemm/README.md`.
- **Chart:** `p9-target-quant.png`.

### 12. `2026-09-22-three-reviewers-roasted-us` — 2026-09-22 13:00
- **Title idea:** "We asked three AIs to tear our blog apart. They found 3 blockers."
- **Sticker:** `3 blockers` — "and a pile of majors"
- **Angle:** the corrections post. Before publishing, three independent reviewers (methodologist, microarchitect, maintainer) read everything. What they caught: prefill peak computed with the Zen 4 rate (Zen 5 does twice the FLOPs per cycle); the MoE's uncounted recurrent state; "most of the draft comes from cache" should be "at least a fifth"; the quality metric re-tokenized text and scored one kernel path; speedups divided by baselines from other sessions; the 57 GB/s "memory controller" claim made without evidence. What changed: noise-floor rule (~5%), bracketed baselines, ranges instead of point numbers, corrections struck through. Tone: gracious, funny, a little humbled. Do not quote reviewers verbatim beyond short phrases; paraphrase.
- **Facts:** F-P1-6, F-P5-3, F-P8-1, F-Q-2, F-P3-1 (range), F-P10-4.
- **Lab:** `docs/adr/0011-iteration-three-rigor.md`, `PROGRESS.md` iteration-three entries, the correction sections in P1/P5/P8 READMEs, `docs/FINDINGS.md`.
- **Chart:** none.

### 13. `2026-09-22-is-our-ram-at-half-speed` — 2026-09-22 15:00
- **Title idea:** "Is our RAM running at half speed? (The Bouncer is suspicious.)"
- **Sticker:** `105 ns` — "memory latency; healthy is ~75"
- **Angle:** cliffhanger. One die already hits the 57 GB/s cap, a second adds nothing, so the Bouncer is the memory controller, not the fabric. The kit has no AMD EXPO profile; boards often halve the controller clock in that case. Latency 105–115 ns fits. Healthy two-die AM5 reads ~75 GB/s. The fix is one BIOS setting (UCLK DIV1 Mode = UCLK=MEMCLK); nobody has checked yet. If it is true, every number on this blog is ~1.3× low and every ratio stands. Invite readers with the same kit to check theirs.
- **Facts:** F-P10-1 … F-P10-4, F-P0-2, F-P0-3.
- **Lab:** `poc/P10-memory-subsystem/README.md`, `results/p10-latency-*.csv`.
- **Chart:** none (a latency table by cache tier from the CSVs is welcome).

## Batch 2

### 14. `2026-09-22-the-intern-beat-the-phd` — 2026-09-22 14:50 (published)
- **Title idea:** "We hired two PhDs to replace the Intern. The Intern won."
- **Sticker:** `2.17× vs 1.89×` — "the Intern vs the best learned head"
- **Angle:** EAGLE-3 heads are the fancy GPU-era drafters: they read the Target's own thoughts (hidden states). Two public ones exist for our Target; neither ran in llama.cpp against Qwen2 because of one missing line; we added it, converted both to GGUF (first time we know of), and raced them against the plain 0.5B Intern. The Intern won: 2.17× vs 1.89× at best. Their acceptance matches what their authors publish, so it is not a bug; they are just weaker drafters for a quantized Target on this workload. Precision of the head changes cost, not acceptance (again). Likely reasons labelled as guesses. Clean session: baseline drifted 0.2%.
- **Facts:** F-P12-1 … F-P12-8, F-P3-4.
- **Lab:** `poc/P12-eagle3-heads/README.md`.
- **Chart:** none yet (table).

### 15. `2026-09-22-eight-threads-four-cores` — 2026-09-22 15:20 (published)
- **Title:** "We hired eight movers. It was four guys in two hats each."
- **Sticker:** `8 = 4` — "threads we pinned vs cores we got"
- **Angle:** confession. `--cpu-range` + `--cpu-strict 1` put 8 threads on 4 hyperthreaded cores; a fact-checker caught it; the same-day rerun overturned the die-split penalty (2%/12%, not 25%), softened the V-cache result (+4%), and left the rest standing. Positive control. Advice on masks.
- **Facts:** F-PIN-1/2, F-P13-1 … F-P13-7, F-P8-2/3, F-P2-3.
- **Lab:** `poc/P13-pinning-rerun/README.md`, `docs/adr/0012-pinning-masks.md`.
- **Chart:** `p13-layouts.png`. Posts 5 and 10 were corrected in place (strike-throughs) and link here.

## Next up (not yet written)

- **The two llama.cpp PRs, filed** (P6 pause-pools, P7 Q4_0 pointer fix, plus the one-line qwen2 `t_layer_inp`): a post when they are actually sent, with maintainer feedback if any.
- **The BIOS check** (P10 cliffhanger): if the memory controller was at half clock, a re-baseline post ("Everything on this blog just got 1.3× faster").
- **Tiled GEMM + `--no-repack` as a verification-curve story**: does the PR flatten t(8+) on Q4_K_M?
- **Reruns queued by the review** (P2 at 3 reps, P6 interleaved stock vs patched).

