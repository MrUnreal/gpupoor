# FACTS.md — the ledger

Every number a post uses comes from here. Each fact has an id, the value, the conditions, the source in the lab notebook (not public) and a confidence note. Posts cite ids on their receipt. When a fact is superseded, strike it and add the new one below it; never delete.

Machine for everything unless noted: AMD Ryzen 9 9950X3D (16 Zen 5 cores, two dies: CCD0 with 96 MB L3 "V-cache", CCD1 with 32 MB), 2 × 48 GB DDR5-6000 (Kingston FURY Renegade, XMP only), Windows 11, llama.cpp commit a894dae built with gcc 14 (WinLibs), `GGML_OPENMP=OFF`, 16 threads, greedy decoding. RTX 5080 present and deliberately unused.

Session noise: plain decode of the same 14B model varied 6.24–6.70 tok/s across one day. Differences under ~5% between sessions are not interpreted.

## Memory and the ceiling (P0, P10)

| id | fact | source | note |
|---|---|---|---|
| F-P0-1 | Spec-sheet bandwidth of 2 × DDR5-6000: **96 GB/s** | arithmetic | |
| F-P0-2 | Measured STREAM read peak: **57 GB/s** (55.8–58 across thread configs) | `results/stream.json`, `bench/stream` | the denominator for every utilization |
| F-P0-3 | One core alone streams **52–54 GB/s** | `results/stream.json` | |
| F-P0-4 | 48 MB working set: **233 GB/s** on the V-cache die vs **72 GB/s** on the other die (8 threads) | `docs/hardware.md` | one run; a later run gave 321 vs 110, same story |
| F-P10-1 | One die alone hits the cap; adding the second die adds nothing | `poc/P10-memory-subsystem` | points at the memory controller, not the fabric |
| F-P10-2 | Random DRAM latency (pointer chase, 4 KB pages): **105–115 ns** | `results/p10-latency-*.csv` | a healthy 1:1 setup reads ~70–80 ns in AIDA64 |
| F-P10-3 | Typical two-die AM5 read bandwidth at DDR5-6000, memory controller 1:1: **~75 GB/s** | published AIDA64 reports | external |
| F-P10-4 | Suspected cause: memory controller at half clock (UCLK = MEMCLK/2) on a non-EXPO kit | P10 README | **unconfirmed**, needs the BIOS; if true every absolute number here is ~1.3× low |

## Dense decode (P1)

| id | fact | source | note |
|---|---|---|---|
| F-P1-1 | 12 dense models from 0.5B to 14B, Q2_K to F16, decode at **90–99%** of the measured ceiling | `results/p1-baseline.jsonl` | ±5% depending on which STREAM cell is the denominator |
| F-P1-2 | Qwen2.5-0.5B at F16 / Q8_0 / Q4_K_M: **54.8 / 102.9 / 136.7 tok/s** for 0.99 / 0.53 / 0.39 GB per token | P1 | quantization buys exactly its byte ratio |
| F-P1-3 | Qwen2.5-14B Q4_K_M decode: **6.3–6.7 tok/s** | P1, P3, P9 | session-dependent |
| F-P1-4 | Qwen2.5-7B Q4_K_M decode vs threads: 1 → 7.7, 2 → 10.8, 4 → 11.7, 8 → 12.6, 16 → 12.9 tok/s | `results/p1-threads-7b-q4km.md` | 8 cores = 98% of 16 |
| F-P1-5 | One core running the Q4_K dot product streams **34 GB/s** of weights | P1 | the kernel, not DRAM, limits one core |
| F-P1-6 | Prefill 7B Q4_K_M 155 tok/s ≈ 2.2 TFLOP/s-equivalent ≈ **40%** of the ~5 TFLOP/s AVX-512 FP32 peak | P1 | corrected; first version said "near peak" |

## Build traps

| id | fact | source | note |
|---|---|---|---|
| F-B-1 | gcc + libgomp (OpenMP) on Windows: Qwen2.5-0.5B decode **13 tok/s** at 16 threads vs **126–133** with ggml's own thread pool; libgomp gets slower as threads are added (70 → 36 → 22 → 13 at 1/4/8/16) | ADR-0002 | |
| F-B-2 | 32 threads (SMT on) with the spin pool: **13 tok/s** on the same model | P1 | |

## Sparsity (P4)

| id | fact | source | note |
|---|---|---|---|
| F-P4-1 | Random gather of ≥1 KiB rows from a 1 GiB matrix, 16 threads: **85–95%** of the ideal 1/density speedup down to 1% density | `results/gather-1GB-16t.csv` | |
| F-P4-2 | At 1% density, 1 KiB rows: 1 thread **35×**, 4 threads **66×**, 16 threads **72×** (ideal 100×) | gather CSVs | |
| F-P4-3 | One thread touching 50% of 1 KiB rows is **slower** than reading them all (0.8×) | gather CSVs | |
| F-P4-4 | Software prefetch: no measurable effect | gather CSVs | hardware prefetcher + parallel misses already win |
| F-P4-5 | 256-byte rows lose 25–35% even at 16 threads | gather CSVs | |

## Verification cost (P2)

| id | fact | source | note |
|---|---|---|---|
| F-P2-1 | Verifying 4 tokens per step costs **1.07–1.14×** one token (Q4_0 1.07, Q4_K_M 1.08, Q8_0 1.14) | `results/p2-batch.jsonl` | 1 run per point; 3-rep rerun queued |
| F-P2-2 | 8 tokens: **1.24–1.35×**; 16 tokens: **1.48–1.74×** (Q4_0 flattest at 16) | P2 | same caveat |
| F-P2-3 | On 8 cores instead of 16: 8 tokens cost **1.60×**, 16 cost **2.25×** | P2 | why splitting the dies loses |

## Speculative decoding (P3, P6, P9)

| id | fact | source | note |
|---|---|---|---|
| F-P3-1 | 14B Q4_K_M target + 0.5B draft, K = 4, idle pools sleeping: **2.0–2.2×** mean over code / prose / list | P3, P9 | 2.02× in one session, 2.19× in another; range quoted |
| F-P3-2 | Same, per prompt: code **~2.5×**, list ~2.1×, prose **~1.5×** | P3 | |
| F-P3-3 | K = 4 speedup by draft: 0.5B Q4_K_M 2.02×, 0.5B Q8_0 2.02×, 0.5B F16 1.70×, 1.5B Q4_K_M 1.80×, 1.5B Q8_0 1.47×, 3B 1.39× | `results/p3s-draft.jsonl` | 1 rep per cell |
| F-P3-4 | Draft precision does not move acceptance: Q4_K_M / Q8_0 / F16 = **51 / 55 / 53%** | P3 | |
| F-P3-5 | K = 16 never beats K = 2; K = 4 best for every draft | P3 | |
| F-P3-6 | N-gram self-speculation on fresh text: **0.95–1.01×** | P3 | nothing to look up |
| F-P3-7 | Qwen3.6-35B-A3B with its own MTP head as draft: **19.4 → 25.4 tok/s (1.31×)** at K = 2 | `results/p3s-mtp.jsonl` | |
| F-P6-1 | llama.cpp defaults (16 target + 16 draft threads, both spinning): **1.58×**; with `--poll 0 --spec-draft-poll 0`: **2.03×**; ms per step 300 → 231 | P3a, P6 | not interleaved; rerun queued |
| F-P6-2 | Patched llama.cpp (draft gets its own pool, idle pool parked), default flags: **2.03×** | P6 | same caveat |
| F-P6-3 | Before the patch, the draft had no thread pool (new OS threads every token) and `--spec-draft-cpu-range/-poll/-prio` were parsed and ignored | P6, code reading | |
| F-P9-1 | 14B **Q4_0** target, same draft: **15.7 tok/s (2.22×)** vs Q4_K_M 14.5 tok/s (2.19×) at K = 4, same session | `results/p9-*.jsonl` | relative gain equal; Q4_0 faster by bytes |
| F-P9-2 | Best single run of the study: Q4_0 target, code prompt, K = 8: **21.3 tok/s from 7.0 plain = 3.0×** | P9 | |

## The V-cache and the Intern (P8)

| id | fact | source | note |
|---|---|---|---|
| F-P8-1 | SmolLM2-135M Q8_0 (145 MB) alone: **495 tok/s** on the V-cache die vs **301** on the other (8 threads each) | `results/p8-residency-135m.md` | 71 GB/s effective > DRAM ceiling: at least a fifth from cache |
| F-P8-2 | Inside speculative decoding (1.7B target): draft on V-cache die **61.9 tok/s** vs draft on small die **62.2**, same core split | P8 | no measurable benefit; no positive control for pinning |
| F-P8-3 | Target on all 16 cores, draft 4 threads: **78.5 tok/s**; every 8-core-target layout 59–62 | P8 | give the Target every core |
| F-P8-4 | A 386 MB draft (40% of a 1 GB target) with 65% acceptance: **0.99×** | P8 | the 145 MB draft (15%) gave 1.59× |

## The Q4_0 crash (P7)

| id | fact | source | note |
|---|---|---|---|
| F-P7-1 | Every Q4_0 model segfaulted on this gcc/mingw build: `vmovdqa %ymm0,0x40(%rsp)` into a 16-byte-aligned slot | `results/raw/q40-diag2.txt` | |
| F-P7-2 | Cause: on SEH targets gcc caps stack alignment at 16 bytes but still emits aligned 32-byte stores for a by-value `__m256i` temporary; gcc bug 54412 (open since 2012); `-mpreferred-stack-boundary=5` is rejected | P7 | newer w64devkit ships a mitigation (`-muse-unaligned-vector-move`) |
| F-P7-3 | Fix: pass the 16-byte lookup table by pointer, build the vector in a register; 18 lines added, 20 removed | P7 patch | PR text ready, not yet filed |
| F-P7-4 | After the fix: 7B Q4_0 prefill **217–243 tok/s**, decode 13.5–14.4 | P7, P11 | |

## Tiled K-quant GEMM (P11, llama.cpp PR #27851)

| id | fact | source | note |
|---|---|---|---|
| F-P11-1 | 7B Q4_K_M prefill, stock: **162 tok/s**; with the PR (repack on): **204**; with the PR and `--no-repack`: **~405** (2.5× stock) | `results/p11-tiled-gemm-clean.md` | 3 reps (no-repack: 2 runs) |
| F-P11-2 | 7B Q6_K prefill: **110 → 391 tok/s** (3.6×) | P11 | |
| F-P11-3 | Q4_0: unchanged (not a tiled type); decode unchanged for every quant | P11 | |
| F-P11-4 | PR's own tests + 1323/1323 CPU matmul backend tests pass on gcc 14 | P11 | |

## The MoE (P5)

| id | fact | source | note |
|---|---|---|---|
| F-P5-1 | Qwen3.6-35B-A3B (3B active) decodes at **19.6 tok/s**; dense 3B models at the same weight bytes: 27.3–27.7 | P1, P5 | 28% gap |
| F-P5-2 | The gap is flat in thread count (20.8 / 21.2 / 20.2 tok/s at 4 / 8 / 16 threads) | `results/p5-threads-moe.md` | not barrier overhead |
| F-P5-3 | Leading hypothesis: 30 of 41 layers carry a 2 MiB F32 recurrent state read and written every token, uncounted in "active bytes" | P5 correction | **hypothesis**, per-op profile not run |

## Quality (ADR-0008)

| id | fact | source | note |
|---|---|---|---|
| F-Q-1 | Greedy decoding is not bit-identical across 8 vs 16 threads or across single-token vs batched kernels | P3a | so "lossless" cannot be checked with string equality |
| F-Q-2 | v1 score (text re-tokenized, batched path): speculative outputs within the plain output's band (−0.21 to −0.24 nats/token, 95–98% argmax agreement) | `results/quality.jsonl` | **flawed method** (reviewers): round-trip tokenization, one path; v2 pending |

## Pending (do not publish as results)

- F-P12: EAGLE-3 learned draft heads for Qwen2.5-14B (two public heads, converted to GGUF here), running.
- Quality v2 (token ids, serial vs batched paths).
- 3-rep reruns of P2 and interleaved P6.
- BIOS check of the memory controller clock.
