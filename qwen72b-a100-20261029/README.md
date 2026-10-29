# Qwen2.5-72B on eight A100 40 GB GPUs

Study completed 29 October 2026. **Native vLLM delivered 9.594 requests/s and 2.559 s p95; disaggregated Dynamo delivered 7.480 requests/s and 3.967 s p95.** Disaggregation had **22.03% lower request throughput and 55.06% higher p95** in this matched full pair. This is evidence for the tested configuration, not a universal conclusion about Dynamo.

This folder is separate from the earlier [Gemma studies](../README.md). It publishes aggregate results only. Customer prompts, images, generated responses, request IDs, internal endpoints, original decks and credentials remain private.

## Workload and hardware

| Item | Measured setup |
|---|---|
| Model | `Qwen/Qwen2.5-72B-Instruct`, BF16; not a VL or AWQ checkpoint |
| Model revision | `495f39366efef23836d0cfae4fbe635880d2be31` |
| Dataset | All 25,648 supplied text rows, original order and duplicates preserved |
| Images | **None**; this was text-only inference |
| Actual token lengths | 740.68 mean input tokens; approximately 48.3 mean output tokens |
| Load | **20 concurrent requests globally**, closed loop; resulting RPS was measured, not fixed at 10 RPS |
| GPUs | **8 × NVIDIA A100-SXM4-40GB**, 40,960 MiB each, NV12 between every GPU pair |
| Software | Same unmodified runtime image and vLLM 0.31.0 for matched native/Dynamo runs; CUDA 13.0, PyTorch 2.13.0+cu130 |
| Sampling | Temperature 0, top-p 1, seed 42, maximum 200 output tokens, natural EOS, streaming |
| Context | 2,560 tokens, sufficient for observed prompt lengths plus output budget |
| Cache preparation | Prefix caching on for the revised comparisons; separate compilation priming, fresh measured workers and matched per-worker warmup |
| Confirmed Qwen production SLA | **Not supplied**; the Gemma latency targets do not apply |

The earlier first-deck reference used 80 GB cards and larger serving layouts. These are **eight-GPU, 40 GB adaptations**, not exact reproductions of the old 16-GPU configurations. No 50% GPU saving at an equivalent production SLA was demonstrated. Sampling and runtime were pinned explicitly because the old invocation details were not fully established.

## Full original-dataset comparison

Both systems had **eight allocated worker GPUs**, processed all 25,648 rows, had zero HTTP errors/timeouts and passed the global-concurrency and request/token-integrity audits. Native used two independent four-GPU model copies. Dynamo used one four-GPU prompt-processing worker and one four-GPU token-generation worker.

| Configuration | N | RPS | Mean s | p50 s | p90 s | p95 s | p99 s | TTFT p95 s | Cap flags | Format failures |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Native, two TP4 copies | 25648 | 9.594 | 2.080 | 2.039 | 2.417 | 2.559 | 2.960 | 0.204 | 38 | 31 |
| Dynamo, TP4 prefill + TP4 decode | 25648 | 7.480 | 2.669 | 2.474 | 3.616 | 3.967 | 4.778 | 2.052 | 35 | 27 |

Latency columns are client full-response times in seconds, through stream finish/usage/EOF. p50 is the median; p90/p95/p99 are per-request percentiles. TTFT means time to first content token. Throughput includes draining the workload. HTTP success, format validity and semantic correctness are separate checks.

The single matched full pair and consistent screening direction support the observed outcome. No repeatability or statistical-equivalence claim is made. Both full runs had cap flags, so neither establishes an entirely untruncated workload result.

## Routing and disaggregation screens

Each screen replayed the same 1,000 real rows at global concurrency 20. These are **screening results**, not full-dataset validations. Rows are sorted by observed p95; differences below 1% do not establish a performance win without repeats.

| Configuration | N | RPS | Mean s | p50 s | p90 s | p95 s | p99 s | TTFT p95 s | Cap flags | Format failures |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Dynamo, two TP4 copies, KV-aware | 1000 | 9.577 | 2.071 | 2.040 | 2.383 | 2.534 | 2.890 | 0.208 | 2 | 2 |
| Dynamo, two TP4 copies, round-robin | 1000 | 9.572 | 2.073 | 2.042 | 2.404 | 2.553 | 2.908 | 0.204 | 0 | 0 |
| Native, two TP4 copies | 1000 | 9.584 | 2.071 | 2.042 | 2.393 | 2.559 | 2.912 | 0.207 | 0 | 0 |
| Dynamo, TP4 prefill + TP4 decode | 1000 | 7.380 | 2.690 | 2.536 | 3.636 | 3.901 | 4.433 | 2.045 | 1 | 1 |

The KV-aware screen's p95 was 2.534 s versus native's 2.559 s, under a 1% difference; request throughput was essentially tied. It also had two cap flags/format failures. No repeatable KV-routing improvement was established. Round-robin Dynamo was also near native performance.

## Cache reuse: present in both systems

The cache-hit fractions below are **reused prompt tokens divided by queried prompt tokens**, from measured-window counter deltas. They are not percentages of requests. Native and Dynamo worker caches both reused substantial prompt content.

| Run | Worker role | Queried tokens | Hit tokens | Token hit rate |
|---|---|---:|---:|---:|
| native / full | native, worker 0 | 9490135 | 7471728 | 78.73% |
| native / full | native, worker 1 | 9506750 | 7454976 | 78.42% |
| disagg / full | prefill, worker 0 | 18996885 | 15027024 | 79.10% |
| disagg / full | decode, worker 1 | 18996885 | 14834544 | 78.09% |
| native / screen | native, worker 0 | 370258 | 291552 | 78.74% |
| native / screen | native, worker 1 | 368881 | 290096 | 78.64% |
| dynamo-kv / screen | backend, worker 0 | 369747 | 289680 | 78.35% |
| dynamo-kv / screen | backend, worker 1 | 369392 | 294160 | 79.63% |
| dynamo-rr / screen | backend, worker 0 | 370290 | 290816 | 78.54% |
| dynamo-rr / screen | backend, worker 1 | 368849 | 290736 | 78.82% |

High cache reuse does not by itself establish additional Dynamo benefit: native local prefix caching already avoided much of the repeated prompt work. Disaggregated prefill and decode counters are listed separately; adding their denominators would count the request through two stages. Cache-aware routing and prefill/decode separation are distinct features.

## What the instrumentation supports

- Native mean GPU utilization was about **99.9%** across all eight GPUs. Disaggregated prefill utilization averaged **43.5%**, while decode averaged **99.5%**. This is consistent with a split that leaves generation constrained while prompt-processing capacity is less occupied.
- Full-pair TTFT p95 was **0.204 s native versus 2.052 s disaggregated**. Sampled decode waiting averaged about 4.84 requests, compared with zero sampled native waiting.
- The report recorded **102,592 rank-level NIXL transfers totaling approximately 1.426 TB**, with zero failed transfers or preemptions. These are rank-level operations, not 102,592 distinct requests.
- These observations do not isolate the individual causal contribution of routing, scheduling, KV transfer or the required KV-layout differences. GPU utilization is not a direct measurement of theoretical compute efficiency.

## Original TP8 baseline, preserved separately

One model copy was split across all eight GPUs with **prefix caching off** before the scope correction. Both topology and cache policy differ from the later two-copy comparison, so the difference cannot be attributed to TP size alone. These rows are not included in the matched ranking above.

| Configuration | N | RPS | Mean s | p50 s | p90 s | p95 s | p99 s | TTFT p95 s | Cap flags | Format failures |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Native, one TP8 copy (cache off) | 1000 | 6.915 | 2.880 | 2.850 | 3.326 | 3.561 | 4.094 | 0.464 | 0 | 0 |
| Native, one TP8 copy (cache off) | 25648 | 6.867 | 2.910 | 2.859 | 3.369 | 3.564 | 4.120 | 0.422 | 35 | 28 |

## Output validity and limitations

- Full native: **25,617/25,648 format-valid**, 38 cap flags. Full disaggregated: **25,621/25,648 format-valid**, 35 cap flags. Zero HTTP errors do not mean zero application errors.
- `cap_hits` preserves the study's conservative reported cap flags. The exact `finish_reasons` are retained in JSON; a flag and a length finish reason are distinct observations. Capped/truncated results are not promoted as unrestricted natural-completion measurements.
- No ground-truth labels or blinded semantic evaluation were available. Format checks or output agreement establish neither correctness nor semantic equivalence.
- No output length was forced to the simulator's assumed 48 tokens. Real output distributions were measured, and all comparisons retained the same 200-token limit.
- The published image had as-shipped dependency-check conflicts, retained in private setup evidence. Empirical model, NCCL and transfer checks are not a blanket production-support certification.
- Screens and full passes remain separate. No image workload, traffic-skew sweep, regrouped dataset or 16-GPU baseline was measured in this study.

## Relationship to the two supplied decks

The first deck concerns native text-model replicas versus Dynamo disaggregation. This study independently finds the same direction for the adapted 72B configuration; absolute numbers and GPU-saving claims cannot be carried across the hardware/software/layout differences.

The second deck concerns Qwen2.5-VL AWQ models and KV-aware routing under specific text/image and traffic-distribution settings. Its model, precision, replica counts and arrival patterns were not reproduced here. The short BF16 routing screen does **not** validate or invalidate the reported large second-deck benefit.

To reproduce that comparison, obtain the original deployment/benchmark scripts, exact model revision, input/request files, load settings, traffic-skew construction and GPU/replica allocation. Any new test should be a separate workload series with matched native and Dynamo cache preparation and output validation.

## Files, provenance and runtime

- [summary.csv](summary.csv): flat aggregate metrics for all eight measured runs, including p50/p90/p95/p99, TTFT, estimated output-token timing, throughput and validity counts.
- [summary.json](summary.json): precise aggregate statistics, model/runtime pins, per-worker cache counters and GPU observations. Source paths identify private artifacts, not files committed here.
- [SLACK_UPDATE.txt](SLACK_UPDATE.txt): proposed update; not sent automatically.

The source summary's SHA256 is recorded in JSON. The original CSV SHA256 is `f0fa5f2d47b3c2b639f4b65ccbf9fe56c5d27d7318498d5c55554f28d07cc7dc`; original data was preserved.

Pinned runtime image:

```text
nvcr.io/nvidia/ai-dynamo/vllm-runtime@sha256:e01aaacd4d8aa571ad960f541eafc1402f4b94a1abcdd8e2745e7e3408c84ae0
```

The report accounts for **3.914 hours before final backup verification** and **30.243 allocated GPU-hours**. The agent reported **4.01 hours through verified backup completion**. Dedicated-instance exposure was 31.311 GPU-hours, including idle time during that report window. No actual instance price was supplied, so dollars/request and cost savings were not calculated.

Private source records, scripts, manifests, logs and the verified backup remain outside this public folder. The agent reported extraction verification of 11,689 files with zero mismatches and stopped the study workers/monitors. VM shutdown and off-instance backup retention are separate operations.
