# Adobe Gemma inference benchmark results

As of **29 October 2026**. These are hardware measurements on **NVIDIA RTX PRO 6000 Blackwell Server Edition GPUs, 96 GB each**, matching the GPU family intended for AWS G7e.

**Best Gemma 4 text result: two native one-GPU copies, approximately 654–668 ms p95 at 10 requests/second total.** This misses the confirmed 200–300 ms target. Dynamo did not give a repeatable latency improvement in the matched text comparisons. The strongest VLM result at total concurrency 32 used two native copies: **12.058 s p95 and 3.432 completed requests/second**; no VLM SLA has been confirmed.

## Workloads and SLA

| Workload | What it does | Measured workload/load | Confirmed full-response p95 target | Outcome |
|---|---|---|---|---|
| Gemma 3 4B text | Scores and rates text | 25,648 requests; 3,000-request screens; 50 RPS total | 150 ms and 300 ms | Neither met; fastest repeated full passes about 360 ms with MXFP4, with decision differences |
| Gemma 4 text | Writes image prompts from text; it does not generate images | 991 original requests at 10 RPS total | 200–300 ms | Unmet; best observed p95 about 654 ms |
| GenEdit-100 VLM | Reads an image and instruction; generates structured JSON | 100 image requests; total client concurrency 1–32 | Not provided | Latency/throughput characterized; no SLA verdict |
| GDR VLM | Separate supplied image workload | 887-image subset available; not benchmarked | Not confirmed | No result |

**Why 360 ms became 900 ms:** these are different models and workloads. Gemma 3 scoring used about 738 input / 53 output tokens; Gemma 4 text used about 6,134 input / 63 output tokens. This was a new service baseline, not a regression of the same test.

## How to read the results

- **Full-response latency** measures HTTP request send to completed response, including all generated text. **p50** is the median; **p90/p95/p99** mean 90%/95%/99% of measured requests finished within that time.
- **TTFT** is time to first token. Gemma 4 text requests were non-streaming, so TTFT was **not measured**. VLM TTFT uses the first nonempty streamed content. **TPOT** is the streaming client's estimated time per output token after the first.
- **TP1** means one full model on one GPU. **TP2** means one model split across two GPUs. **Two TP1 copies** means two full models running simultaneously, with round-robin requests split between them.
- **MTP2/MTP4** use an assistant model to propose two/four tokens for the target to verify. **FP8/MXFP4** use reduced numerical precision; speed does not establish output-quality equivalence.
- **RPS** means requests per second. Text arrival rates are total service load, not per GPU. Completed RPS includes drain time and is slightly below dispatch RPS. VLM uses fixed concurrency, not a fixed arrival rate.
- Tables are sorted by ascending **per-run p95**, separately by workload, runtime, sample size and VLM concurrency. This is a descriptive ordering, not proof that every small difference is significant. Repeated measurements appear separately; percentiles are never averaged or pooled.
- **—** means unavailable. All latency columns are **milliseconds** except the explicitly labeled VLM tables, which use **seconds**. Startup failures, partial-workload runs and distorted profiles are excluded from successful-run rankings.

## Gemma 4 text: current runtime, vLLM 0.31.0

All rows use the same published Dynamo 1.6.0 vLLM runtime image, FP8 target, FP8 KV cache and MTP4. Native rows use native vLLM in that image. Each row replays all 991 unchanged requests at 10 RPS total. TTFT is unavailable for every row. All ranked runs had **zero HTTP failures and zero truncations**; application-format failures are separate.

Run names and experiment IDs map directly to `source` paths in [results.json](results.json). Labels below explain what changed.


| Configuration / pass | GPUs | Mean | p50 | p90 | p95 | p99 | Completed RPS | Format failures / 991 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Two TP1 copies, MTP4; text-only flag — `control` (overnight) | 2 | 515.3 | 533.7 | 627.4 | 654.3 | 721.4 | 9.960 | 25 |
| Two TP1 copies, MTP4; router comparison — `native_rr` (native-vs-dynamo2-v31) | 2 | 519.9 | 534.0 | 631.5 | 663.1 | 753.7 | 9.960 | 23 |
| TP2 vs two TP1 copies, MTP4 — `dual` (replica-vs-tp2-v31) | 2 | 519.4 | 534.4 | 635.3 | 668.2 | 742.9 | 9.963 | 22 |
| Two TP1 copies, MTP4; router comparison — `dynamo_rr_warm` (native-vs-dynamo2-v31) | 2 | 526.0 | 541.8 | 642.9 | 671.0 | 768.9 | 9.960 | 22 |
| Two TP1 copies, MTP4; router comparison — `dynamo_rr_repeat` (native-vs-dynamo2-v31) | 2 | 527.1 | 542.3 | 646.3 | 678.7 | 784.1 | 9.954 | 25 |
| Two TP1 copies, MTP4; text-only flag — `lmonly` (overnight) | 2 | 520.8 | 535.8 | 638.6 | 679.5 | 779.5 | 9.958 | 27 |
| TP2 vs two TP1 copies, MTP4 — `tp2` (replica-vs-tp2-v31) | 2 | 576.1 | 577.2 | 727.6 | 755.7 | 842.9 | 9.966 | 28 |


### What the current text tests showed

| Experiment | What changed | Result / interpretation |
|---|---|---|
| Two copies vs TP2 | Same two GPUs; duplicate the model instead of splitting one copy | 668 vs 756 ms p95 in the matched topology test, an 11.6% reduction. Fresh two-copy native controls later measured 663 and 654 ms. This is a layout gain, not a Dynamo gain. |
| Dynamo vs native TP2 | Add Dynamo aggregated serving around the matched TP2 engines | Native 756/752 vs Dynamo 756/754 ms p95; no repeatable gain. |
| Dynamo vs native two copies | Hold the two-copy layout fixed; compare round-robin routing | Native 663 vs Dynamo 671/679 ms p95; no gain. Both had 94.44% prefix-token hit rate and balanced workers. |
| Language-model-only flag | Disable vision capability for the text-only workload | 679 vs 654 ms control p95; no gain. |

The **654 ms best observation is 654 ÷ 300 ≈ 2.18 times the upper latency target**. None of these configurations qualifies for the text SLA. Using two GPUs improves latency but does not establish GPU-cost savings or maximum sustainable capacity.

## Gemma 4 text: original runtime, vLLM 0.29.0

All ranked rows again use 991 requests at 10 RPS total. These are historical experiments and must not be used as matched comparisons against v0.31.0: other image dependencies changed too. The initial Dynamo comparison retained v0.29.0 even though Dynamo 1.6.0's optional backend declared v0.31.0; that combination was experimental.


| Configuration / pass | GPUs | Mean | p50 | p90 | p95 | p99 | Completed RPS | Format failures / 991 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Native TP2, MTP4; smaller batch cap — `control_b` (cap2496) | 2 | 564.5 | 570.5 | 692.3 | 737.4 | 859.6 | 9.965 | 18 |
| Native TP2; MTP2 vs MTP4 — `mtp4_c` (mtp4) | 2 | 567.9 | 573.7 | 690.6 | 739.0 | 837.7 | 9.964 | 18 |
| Native TP2, MTP4; tail diagnostic — `control_a` (tail-trace) | 2 | 563.0 | 568.9 | 687.3 | 742.3 | 837.7 | 9.966 | 17 |
| Native TP2; MTP2 vs MTP4 — `mtp4_b` (mtp4) | 2 | 565.1 | 567.8 | 691.1 | 748.1 | 854.1 | 9.965 | 17 |
| Native TP1 vs TP2, MTP2 — `tp2_c` (tp2) | 2 | 586.2 | 591.0 | 728.0 | 749.2 | 838.6 | 9.969 | 27 |
| Native TP2, MTP4; smaller batch cap — `candidate_a` (cap2496) | 2 | 567.8 | 573.4 | 701.2 | 749.7 | 843.2 | 9.960 | 19 |
| Native TP2; MTP2 vs MTP4 — `mtp2_a` (mtp4) | 2 | 586.3 | 589.1 | 718.1 | 750.0 | 842.3 | 9.957 | 27 |
| Native TP1 vs TP2, MTP2 — `tp2_b` (tp2) | 2 | 589.5 | 598.7 | 726.8 | 750.1 | 856.6 | 9.965 | 27 |
| Native TP2, MTP4; smaller batch cap — `candidate_c` (cap2496) | 2 | 566.3 | 571.5 | 700.7 | 752.3 | 854.3 | 9.964 | 19 |
| One-GPU native/Dynamo, MTP2 — `native_a2` (dynamo-v31) | 1 | 568.2 | 572.8 | 698.4 | 752.4 | 871.9 | 9.952 | — |
| One-GPU native/Dynamo, MTP2 — `dynamo_b2` (dynamo-v31) | 1 | 569.3 | 573.5 | 710.7 | 754.2 | 850.7 | 9.965 | — |
| One-GPU native/Dynamo, MTP2 — `native_a` (dynamo-v31) | 1 | 576.4 | 581.8 | 713.2 | 756.0 | 839.3 | 9.966 | — |
| One-GPU native/Dynamo, MTP2 — `dynamo_b` (dynamo-v31) | 1 | 573.4 | 574.8 | 701.6 | 756.2 | 866.9 | 9.956 | — |
| One-GPU native/Dynamo, MTP2 — `native_a2` (dynamo) | 1 | 704.9 | 729.7 | 863.4 | 886.0 | 975.8 | 9.956 | — |
| One-GPU native/Dynamo, MTP2 — `dynamo_b2` (dynamo) | 1 | 705.1 | 724.4 | 860.5 | 887.2 | 966.8 | 9.959 | — |
| Native TP1, MTP2; MoE kernel tiles — `candidate_2` (moe) | 1 | 707.4 | 732.9 | 864.5 | 887.6 | 960.5 | 9.956 | 26 |
| Native TP1, MTP2; MoE kernel tiles — `default_1` (moe) | 1 | 703.7 | 728.2 | 858.6 | 888.5 | 968.1 | 9.955 | 28 |
| One-GPU native/Dynamo, MTP2 — `dynamo_b` (dynamo) | 1 | 710.5 | 733.3 | 870.9 | 890.2 | 984.6 | 9.953 | — |
| One-GPU native/Dynamo, MTP2 — `profile_native_unprofiled` (dynamo) | 1 | 705.8 | 732.1 | 862.0 | 892.3 | 957.7 | 9.952 | — |
| Native TP1 vs TP2, MTP2 — `tp1_a` (tp2) | 1 | 713.2 | 737.9 | 868.2 | 907.2 | 1,004.1 | 9.953 | 16 |
| Native TP1, MTP2 baseline — `baseline-20261027T175006Z` (baseline) | 1 | 715.5 | 735.3 | 868.1 | 914.2 | 979.9 | 9.955 | 28 |
| One-GPU native/Dynamo, MTP2 — `native_a` (dynamo) | 1 | 712.6 | 736.5 | 870.9 | 927.5 | 971.5 | 9.956 | — |
| Native TP1 vs TP2, MTP2 — `tp1_d` (tp2) | 1 | 717.0 | 738.1 | 872.9 | 928.7 | 990.1 | 9.955 | 25 |


### What the original text tuning showed

| Experiment | Method | Result |
|---|---|---|
| Initial baseline | One GPU, FP8, MTP2 | 914 ms p95; 991/991 HTTP successes, but 28 strict-format failures. |
| Dynamo one GPU | Two matched native/Dynamo pairs | 927 vs 890 ms, then 886 vs 887 ms p95; first apparent gain did not repeat. |
| MoE kernel tiles | Load an available same-shape Triton configuration | Warmed candidate 888 vs 889 ms control p95; no material gain. Short trace also showed no kernel-time gain. |
| TP2 | Split target and assistant over two GPUs | 749–750 vs 907–929 ms p95 in matched controls. Custom all-reduce failed at startup, so measured controls and candidates used the NCCL fallback. |
| MTP4 | Increase speculative tokens from two to four on TP2 | Mean improved 18–21 ms; p95 only 2–11 ms. Accepted tokens per verification round rose from 1.266 to about 1.84. |
| Smaller batch-token cap | 12,288 → 2,496 tokens | Candidates 750/752 vs 737 ms control p95; no gain. The proposed 1,024 cap failed startup. |
| Tail analysis and trace | Examine recurring slow IDs and shared GPU batches | Decode dominated; overlapping prompt work contributed to a small slow cohort. Even ideal removal of its slowdown reduced one control p95 by only 9.4 ms. |

The initial profile measured roughly **54 ms prefill** (reading the prompt), **563 ms decode** (generating text) and **89 ms not reliably separable** with the instrumentation. Later two-copy decode means were approximately 390 ms. These are mean counters, not additive p95 components. Dynamo routing did not remove this generation work; worker balance, cache reuse and negligible engine queueing left little demonstrated routing headroom. This does not establish that Dynamo cannot benefit other workloads or higher loads.

## GenEdit-100 VLM: vLLM 0.31.0

Each row uses all 100 original images and instructions, BF16 target/assistant, MTP4, a 560-token vision budget and constrained JSON. All native rows had **100/100 strict-schema-valid responses, no HTTP/parse errors and no truncations**. Schema validity does not establish semantic correctness.

Rank **within the same total concurrency**. Increasing concurrency changes the load: it raises throughput while making individual requests slower. The two-copy comparisons also change CPU partition within the same total server CPU pool.


| Total concurrency | Layout | Mean s | p50 s | p90 s | p95 s | p99 s | Completed RPS |
|---:|---|---:|---:|---:|---:|---:|---:|

| 1 | One GPU | 2.994 | 2.766 | 4.421 | 4.733 | 7.409 | 0.321 |

| 4 | One GPU | 4.370 | 4.118 | 6.161 | 6.743 | 10.364 | 0.901 |

| 8 | One GPU | 5.536 | 5.170 | 7.930 | 8.744 | 14.687 | 1.392 |

| 16 | Two one-GPU copies | 5.683 | 5.162 | 8.204 | 9.809 | 14.897 | 2.538 |

| 16 | One GPU | 7.393 | 6.862 | 10.100 | 11.758 | 18.073 | 1.965 |

| 32 | Two one-GPU copies | 7.580 | 7.302 | 10.194 | 12.058 | 18.938 | 3.432 |

| 32 | One GPU | 11.556 | 10.662 | 17.504 | 19.593 | 25.163 | 2.451 |


### VLM first-token and generation timing

| Total concurrency | GPUs | TTFT mean s | TTFT p50 s | TTFT p90 s | TTFT p95 s | TTFT p99 s | TPOT mean ms | Output tokens mean |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|

| 1 | 1 | 0.504 | 0.422 | 0.625 | 1.204 | 1.779 | 3.91 | 636.8 |

| 4 | 1 | 0.641 | 0.541 | 0.879 | 1.299 | 2.218 | 5.91 | 630.7 |

| 8 | 1 | 0.828 | 0.677 | 1.275 | 1.828 | 3.254 | 7.35 | 639.2 |

| 16 | 2 | 0.843 | 0.684 | 1.673 | 1.867 | 2.427 | 7.49 | 643.1 |

| 16 | 1 | 1.109 | 0.818 | 2.528 | 2.711 | 3.262 | 9.90 | 634.0 |

| 32 | 2 | 1.576 | 0.907 | 3.724 | 3.994 | 4.192 | 9.52 | 636.5 |

| 32 | 1 | 3.027 | 1.660 | 8.016 | 8.939 | 9.317 | 13.70 | 628.0 |


### Original VLM baseline, vLLM 0.28.0

| GPUs / concurrency | Mean s | p50 s | p90 s | p95 s | p99 s | TTFT p95 s | Completed RPS |
|---|---:|---:|---:|---:|---:|---:|---:|
| 1 / 1 | 3.130 | 2.867 | 4.736 | 5.235 | 7.533 | 1.433 | 0.307 |


The original package reference used two A100 GPUs, TP2 and in-process calls. Our baseline used one Blackwell GPU and HTTP streaming, so this is not a direct comparison with that reference. The newer v0.31.0 c1 control belongs to a separate runtime series.

VLM latency starts at HTTP send and excludes image preparation and client waiting before send; reported completed throughput includes the client's wall-clock work. Images were RGB-converted and PNG-reencoded as required, without resizing. The 100-request bursts do not establish steady-state maximum capacity. Two copies at c32 reduced p95 by 38.5% versus one GPU, but also used twice as many GPUs.

## Gemma 3 4B text: original 50-RPS study

These tables concern `google/gemma-3-4b-it`, not the larger Gemma 4 service. Original runtime: Dynamo 1.5.1 / vLLM 0.28.0. Full-response latency includes stream completion and usage; TTFT is measured separately. All ranked rows completed without HTTP errors or truncation. Neither the 150 ms nor 300 ms p95 target was met.

### Repeated full-dataset MXFP4 validation — 25,648 requests per pass

Two independent one-GPU copies, MXFP4/Marlin, natural response completion. All five passes together completed 128,240 requests successfully. Differences between native and Dynamo were within measured noise. Reduced-precision decision differences remain unresolved.


| Configuration / pass | GPUs | Mean | p50 | p90 | p95 | p99 | TTFT p95 | Completed RPS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `dynamo2-mxfp4-final-r2` | 2 | 276.6 | 269.6 | 330.7 | 359.3 | 429.5 | 27.2 | 49.970 |
| `vllm2-mxfp4-final-r1` | 2 | 276.9 | 269.7 | 330.5 | 359.5 | 428.6 | 26.3 | 49.970 |
| `dynamo2-mxfp4-final-r1` | 2 | 276.9 | 269.7 | 331.1 | 360.2 | 429.8 | 27.2 | 49.976 |
| `vllm2-mxfp4-final-r3` | 2 | 277.1 | 270.0 | 331.4 | 360.2 | 430.4 | 26.2 | 49.976 |
| `vllm2-mxfp4-final-r2` | 2 | 277.1 | 269.8 | 331.2 | 360.2 | 430.1 | 26.2 | 49.976 |


### Original screening — 3,000 requests per configuration

Name key: `vllm` = native; `dynamo` = Dynamo serving; `1` = one GPU; `2` = two independent one-GPU copies; `tp2` = one copy split across two GPUs; `kv` = KV-aware routing; `ngram` = n-gram speculative decoding. `base` used the initial cache/scheduler settings; `tuned` changed prefix caching, graph/batching settings as a bundle. Unmarked `base/tuned` precision is BF16. Base configurations completed but built backlog, so their completion rates fell below the offered 50 RPS.


| Configuration / pass | GPUs | Mean | p50 | p90 | p95 | p99 | TTFT p95 | Completed RPS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `dynamo2-mxfp4-kv` | 2 | 281.0 | 273.6 | 335.3 | 362.4 | 428.1 | 28.4 | 49.817 |
| `dynamo2-mxfp4` | 2 | 281.1 | 273.4 | 340.0 | 362.6 | 444.7 | 30.1 | 49.756 |
| `vllm2-mxfp4` | 2 | 281.5 | 273.3 | 340.5 | 362.8 | 446.8 | 26.8 | 49.757 |
| `dynamo2-fp8` | 2 | 316.6 | 309.2 | 378.2 | 404.4 | 448.8 | 42.4 | 49.715 |
| `vllm2-fp8` | 2 | 317.6 | 310.2 | 383.7 | 405.9 | 455.1 | 24.3 | 49.715 |
| `dynamo-tp2-fp8` | 2 | 317.0 | 308.4 | 385.2 | 408.8 | 448.1 | 43.2 | 49.780 |
| `vllm-tp2-fp8` | 2 | 318.7 | 309.8 | 386.4 | 411.4 | 449.8 | 26.7 | 49.761 |
| `dynamo1-fp8` | 1 | 344.3 | 336.9 | 409.0 | 437.2 | 487.3 | 30.3 | 49.749 |
| `vllm1-fp8` | 1 | 344.4 | 336.5 | 410.8 | 438.3 | 487.6 | 26.0 | 49.702 |
| `dynamo1-mxfp4` | 1 | 354.7 | 345.3 | 428.9 | 464.7 | 552.5 | 41.9 | 49.732 |
| `vllm1-mxfp4` | 1 | 356.6 | 347.1 | 430.6 | 465.3 | 561.8 | 30.1 | 49.732 |
| `vllm2-mxfp4-ngram` | 2 | 341.0 | 327.2 | 430.0 | 468.4 | 588.4 | 26.9 | 49.809 |
| `dynamo2-mxfp4-ngram` | 2 | 344.6 | 329.9 | 431.9 | 478.3 | 594.0 | 30.0 | 49.655 |
| `vllm2-tuned` | 2 | 448.9 | 437.9 | 538.6 | 571.2 | 641.6 | 31.6 | 49.558 |
| `dynamo2-tuned` | 2 | 448.2 | 437.3 | 540.0 | 573.1 | 641.8 | 42.0 | 49.557 |
| `dynamo1-tuned` | 1 | 553.8 | 540.9 | 665.0 | 709.2 | 787.0 | 41.5 | 49.523 |
| `vllm1-tuned` | 1 | 555.3 | 542.3 | 665.1 | 710.2 | 790.2 | 39.2 | 49.522 |
| `vllm1-base` | 1 | 5,080.5 | 5,117.2 | 7,193.1 | 7,534.3 | 8,090.9 | 5,002.4 | 45.401 |
| `dynamo1-base` | 1 | 5,112.4 | 5,137.8 | 7,248.1 | 7,593.4 | 8,186.8 | 5,055.6 | 45.360 |


### Replacement-VM follow-up — 3,000 requests per pass

All configurations below use two independent native one-GPU copies. `bf1/bf2` are BF16; `mx1/mx2` are MXFP4; `fp8-ref/r2…r7` are FP8 controls and compile-state repeats; `fp8-c1/c2` switch attention to FlashInfer; `fp8-cga/cgb` raise CUDA-graph capture coverage to 2,048 tokens. R1 (`ref`) and R4 compiled fresh; other controls reused compiled artifacts. Sort order alone does not make small kernel-test differences meaningful.


| Configuration / pass | GPUs | Mean | p50 | p90 | p95 | p99 | TTFT p95 | Completed RPS |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `rvm-vllm2-mx2` | 2 | 280.0 | 272.3 | 338.6 | 360.3 | 440.6 | 27.4 | 49.758 |
| `rvm-vllm2-mx1` | 2 | 280.3 | 272.0 | 339.2 | 361.7 | 445.5 | 27.6 | 49.821 |
| `rvm-vllm2-fp8-c2` | 2 | 311.5 | 304.8 | 379.1 | 397.9 | 444.2 | 28.2 | 49.789 |
| `rvm-vllm2-fp8-ref` | 2 | 314.8 | 307.9 | 377.4 | 399.0 | 447.2 | 24.9 | 49.715 |
| `rvm-vllm2-fp8-cga` | 2 | 315.2 | 307.8 | 379.4 | 399.7 | 449.5 | 24.8 | 49.709 |
| `rvm-vllm2-fp8-r4` | 2 | 315.2 | 308.0 | 378.4 | 400.2 | 455.0 | 24.9 | 49.779 |
| `rvm-vllm2-fp8-c1` | 2 | 315.3 | 308.3 | 380.8 | 400.6 | 445.7 | 28.7 | 49.789 |
| `rvm-vllm2-fp8-r2` | 2 | 316.6 | 309.0 | 380.4 | 402.3 | 449.8 | 25.0 | 49.709 |
| `rvm-vllm2-fp8-r7` | 2 | 316.2 | 308.9 | 379.4 | 403.0 | 448.1 | 24.8 | 49.708 |
| `rvm-vllm2-fp8-cgb` | 2 | 315.2 | 307.5 | 379.2 | 403.5 | 450.6 | 24.9 | 49.716 |
| `rvm-vllm2-fp8-r3` | 2 | 315.8 | 308.4 | 378.9 | 403.8 | 449.4 | 24.8 | 49.710 |
| `rvm-vllm2-fp8-r5` | 2 | 316.2 | 308.1 | 382.0 | 404.3 | 450.5 | 24.8 | 49.714 |
| `rvm-vllm2-fp8-r6` | 2 | 317.0 | 309.5 | 381.1 | 406.8 | 456.4 | 24.9 | 49.709 |
| `rvm-vllm2-bf1` | 2 | 448.4 | 437.5 | 537.0 | 569.9 | 641.7 | 32.4 | 49.555 |
| `rvm-vllm2-bf2` | 2 | 448.6 | 437.3 | 538.5 | 571.8 | 639.9 | 32.6 | 49.559 |


### Gemma 3 optimization and quality conclusions

| Change | Result |
|---|---|
| Prefix caching, CUDA graphs and batching bundle | Large improvement over the original overloaded BF16 baseline; individual contributions were not isolated. |
| Two independent copies | Best repeatedly validated layout for this workload. |
| FP8 | About 404–406 ms p95 with two copies; still above SLA. About 9% of SCORE and 5% of RATING decisions differed from BF16 in the later comparison. |
| MXFP4 / Marlin | About 360 ms p95; still above SLA. On repeat-stable answers, SCORE differed on 626/2,995 (20.9%) and RATING on 402/2,970 (13.5%) versus BF16. |
| TP2 | About 409–411 ms p95 in FP8 screening; no gain over two independent FP8 copies. |
| N-gram speculation | Slower: 468–478 ms p95 in MXFP4; no win. |
| KV-aware routing | About 362 ms in screening; no material gain over simple two-copy routing. |
| FlashInfer attention | About 398–401 ms FP8 p95, within reference variation; decision agreement fell outside the warmed reference band. |
| CUDA-graph capture to 2,048 | About 400/403 ms versus 403 ms adjacent control; no meaningful gain, outputs consistent with reference variation. |
| Compile-state pilot | Fresh/reused launches showed output differences; warmed comparisons were used subsequently. Cause was not established. |

Disagreement is **not proof of worse accuracy**: there were no ground-truth labels or blinded quality review. BF16 is a precision reference, not a correctness oracle. Exact wording, HTTP success, JSON validity and semantic correctness are different checks. Neither FP8 nor MXFP4 is established as semantically equivalent to BF16.

## Failed, incomplete and diagnostic experiments — not ranked

| Experiment | Outcome | Why no successful configuration ranking |
|---|---|---|
| Gemma 3 disaggregated FP8 | 1,256/3,000 successes; 1,744 failures/timeouts | Incomplete workload; success-only latency would be misleading. |
| Gemma 3 disaggregated IPv4 retry | 2,943/3,000 successes; 57 failures/timeouts | Incomplete workload; no valid disaggregated win. |
| Gemma 3 PyTorch FP8 linear backend | Engine chose an unintended fallback; stopped before timed tests | Intended kernel did not engage. |
| Gemma 3 270M draft model | Startup assertion on mixed KV-cache groups | No acceptance or timing measurements; draft-only override unsupported in the pinned build. |
| Gemma 3 profile capture | Trace-save stall distorted full-run timing | Diagnostic only; excluded from latency ranking. |
| Gemma 3 252-request pilot | Completed; p95 about 579 ms | Smaller pilot sample, not a 3,000-request comparison. |
| Gemma 4 MoE candidate first pass | 2,464 ms p95, then 888 ms after warming | First-use latency spike; retained in aggregate data but excluded from warmed ranking. |
| Gemma 4 initial two-copy Dynamo pass | 4,330 ms p95 | Measured-window kernel compilation; corrected warmed passes used instead. |
| Gemma 4 tail trace | 25,817 ms p95 with a 27-second trace-write stall | Heavily distorted; diagnostic evidence only. |
| Gemma 4 1,024-token batch cap | Startup failed on 2,496-token multimodal reservation | No measured replay. The subsequent 2,496-token candidate did run and did not help. |
| VLM Dynamo initial media path | Prompt tokens differed from native | Preprocessing mismatch; not a matched comparison. |
| VLM Dynamo worker-side image processing | 94/100 successes, then 95/100 after documented URL/body limit increases | Original large images exceeded frontend/internal transport limits, including a 32 MiB internal message limit. No full-workload latency comparison. |
| VLM TP2 | Smoke and 64-request compilation prime succeeded; fresh measurement startup stalled | No timed 100-request result; prime is not a benchmark. |
| GDR VLM | 887-image subset not staged for these GPU tests | Not run. |

Aggregated serving keeps prompt processing and generation in the same worker; disaggregated serving separates them. The successful Gemma 4 Dynamo comparisons used **aggregated** serving. No Gemma 4 disaggregated performance result was established. The failed Gemma 3 experiments do not establish a general disaggregation limit.

## Measurement boundaries and reproducibility

- Matched text comparisons preserved original request bodies, model revisions, sampling and per-worker warmup. Compile artifacts were primed separately where documented; request/prefix caches were reset through fresh workers and matched warmup. The initial baseline explicitly used warm-prefix cache. These are **warmed** results, not cold-start SLAs.
- Gemma 4 text sampling: temperature 0.7, top-p 0.9, top-k 200, seed 42, max output 512, non-streaming. Gemma 3 used temperature 0, top-p 1, max output 2,048, natural EOS. GenEdit used the package's greedy generation, constrained JSON and max output 3,072. Output lengths varied and are included in aggregate data; equal-length checks supported the larger topology gains, not semantic equivalence.
- The common v0.31.0 image was `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.6.0`, image ID `sha256:c957d97ccb8b225d7671f39afc6b340d910f0c58f8d17837cce6078c77344cd0`. Its as-shipped dependency-check conflicts remain documented in the private compatibility report. The benchmark is not a production qualification.
- Gemma 4 text target: `RedHatAI/gemma-4-26B-A4B-it-FP8-dynamic`, revision `ed35d7abe5d940da41b4ff06eb482feb0be8cb44`. VLM used stock BF16 `google/gemma-4-26B-A4B-it`, revision `4d7ae4984b7db7de8f8457170b3f1a419ee76d52`. Assistant revision: `6e5aaaf4c42b98394530b8fda2e95cadd65c151c`. Gemma 3 target revision: `093f9f388b31de276ce2de164bdc2081324b9767`.
- [results.json](results.json) contains the exact aggregate numbers and original relative experiment/run paths. Text p90 was recomputed from saved numeric timings using linear interpolation; recomputed p95 matched the saved canonical summaries. Gemma 3 uses `request_end_ns − request_start_ns`, rather than mixing in AIPerf's shorter request-latency boundary. Failure-only/partial comparisons were not reconstructed as valid measurements.
- Original detailed reports, raw records, commands, plans and verified backups remain private. This repository contains only aggregate results and source-archive checksums; **customer prompts, images, output text, per-request IDs and credentials are excluded**. Relative source paths identify private artifacts and are not links to files in this repository.

## Current conclusions and remaining work

1. No tested Gemma 3 or Gemma 4 text configuration meets its confirmed full-response p95 SLA. More GPUs did not demonstrate a cost-saving configuration; reducing output length or changing model/precision requires task-owner agreement and quality validation.
2. The clearest Gemma 4 text gain came from serving layout: two independent copies outperformed one TP2 copy. MTP4 helped mean latency modestly. Kernel tiles, smaller batches and the text-only flag did not improve the target.
3. Dynamo has no demonstrated repeatable text-latency gain at the measured load. VLM Dynamo performance remains unmeasured on the complete workload because of media/transport compatibility failures.
4. Before another VLM comparison, preserve preprocessing parity and support all original image sizes. Then measure fresh matched native/Dynamo runs at the same concurrency. VLM SLA and production arrival load still need confirmation; semantic quality needs a separate review.

