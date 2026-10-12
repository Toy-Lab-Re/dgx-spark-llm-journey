# Benchmark Method — Conditions, Output Phases, and Validity

## Provenance and Metric

These tables reproduce numeric summaries from the public-safe handoff audited on 2026-10-12. **VERIFIED** refers to original evidence identified by that auditor; this Windows documentation task did not rerun tests or re-audit original JSON. Private evidence indexes and raw artifacts are excluded.

Hardware: one DGX Spark / GB10, 128 GB unified memory. All reported throughput tests are **single stream, concurrency 1**. The handoff defines streaming decode throughput as completion tokens divided by elapsed time from the first token to completion; it excludes TTFT. Exact first-token counting and stream-event timing cannot be independently recomputed from the handoff alone.

The handoff reports temperature 0, with a temperature-0.6 exception labeled B1–B4 whose mapping is not fully defined in the public report. The BF16/NVFP4 comparison identifies thinking OFF and the cross-model suite identifies thinking ON. Do not assume every table has a fully reconstructed request configuration. Table-specific execution dates, exact prompt hashes, and all input-token counts are not supplied; the overall study spans 2026-09-18 through 2026-10-10.

**Five distinct measurement classes:**

| Class | Meaning | Permitted interpretation |
|---|---|---|
| Thinking decode | Generated reasoning, not a final answer | Reasoning-phase throughput only |
| Answer decode | Output confirmed as answer generation | Answer throughput for that exact configuration |
| Unknown generation phase | Numeric output retained without response text | Decode observation; no answer-performance claim |
| Single-stream test | One generation stream | No multi-user or concurrency claim |
| Synthetic-token test | Artificial token inputs with degenerate output | Invalid as an application-performance benchmark |

單串流是測試規模，不是答案或思考的分類；必須同時標記兩者。不同模型、Runtime、Parser、Prompt、輸出長度與思考設定的數字，不能直接合成加速比。

## 1. BF16 versus NVFP4 on the Same vLLM — VERIFIED

Targets: Qwen/Qwen3.8-27B BF16 and nvidia/Qwen3.8-27B-NVFP4, using vLLM 0.27.1 / NGC 26.08 and the same script. Thinking OFF; decode test output budget 512 tokens.

| Metric | BF16 | NVFP4 |
|---|---|---|
| Decode | 4.40–4.52 tok/s | 12.22–12.40 tok/s |
| TTFT, 20-token prompt | 0.41 s | 0.58 s, after restart |
| TTFT, 1,668-token prompt | 1.72 s | 1.01 s |
| TTFT, 6,528-token prompt | 6.03 s | 2.91 s |
| 27.6k-token needle probes | 2/2; 27.0–27.3 s | 2/2; 13.5–13.6 s |
| Fixed-answer smoke | 7/8 | 7/8, same missed item |
| Reported weight-related memory | 55.31 GiB | 23.47 GiB, including non-torch memory |

**Limits:** KV dtype changed to FP8 and Tool Parser changed alongside weight precision. This is a complete-configuration comparison, not a quantization-only ablation. Memory categories differ in accounting. Eight fixed answers and two needle probes do not prove equivalent quality or establish a general accuracy benchmark. Warm/restart conditions are not uniform.

## 2. 27B NVFP4 with SGLang and DFlash2 — VERIFIED NUMBERS, UNKNOWN OUTPUT PHASE

SGLang 0.5.20; target nvidia/Qwen3.8-27B-NVFP4; auxiliary draft incoai/Qwen3.8-27B-DFlash2; draft block 8 and FP8 KV. The real-text test used thinking enabled by default.

| Workload label | Output tokens | TTFT | Decode | Recorded acceptance length, reported limit 8 |
|---|---|---|---|---|
| Chinese business analysis | 2,491 | 140 ms | 33.35 tok/s | 3.89 |
| English prose | 3,000 | 134 ms | 36.36 tok/s | 4.04 |
| Python code | 3,000 | 137 ms | 42.02 tok/s | 4.92 |

**Limits:** Response text was not retained, so reasoning versus answer generation is NOT FOUND. Exact input lengths and table-specific dates are unavailable. These rows cannot prove business-analysis answer speed, a DFlash2-only gain, or a cross-runtime speedup.

## 3. Cross-model Suite, Thinking ON — VERIFIED NUMBERS

The same script and prompt set compared three complete configurations: 27B NVFP4 / SGLang / DFlash2, Mia Flash-Next / community vLLM / MTP, and RadixArk Flash-Next / SGLang development build / NEXTN / PLE offload.

| Workload label | Output cap | 27B + DFlash2 | Mia Flash-Next + MTP | RadixArk Flash-Next + NEXTN |
|---|---|---|---|---|
| Chinese business analysis | 3,000 | 26.50 tok/s | 22.57 tok/s | 16.87 tok/s |
| English prose | 3,000 | 39.19 | 45.76 | 26.53 |
| Python code | 3,000 | 31.73 | 42.30 | 29.01 |
| SQL generation | 3,000 | 40.00 | 41.20 | 32.65 |
| Tool calling | 1,500 | 51.10 | 48.48 | 40.19 |
| TTFT range | — | 122–207 ms | 275–625 ms | 224–707 ms |

**Output audit:** For the first four rows, the 27B and Mia responses had zero answer characters and spent the 3,000-token budget on reasoning. The table is a thinking-ON configuration comparison, not a cross-model final-answer ranking. The handoff does not provide a complete per-row reasoning/answer split for the RadixArk column or the tool row; do not invent one.

The 42.30 value above is the Mia code-workload cell in this suite. It is unrelated to the coincidentally equal QLoRA gate figure; no QLoRA gate results are imported into this comparison.

Checkpoint/export, engine, draft mechanism, KV choices, and offload differ. Higher acceptance does not isolate a faster engine. Exact request and cache conditions are not completely reproduced in the handoff.

## 4. RadixArk Flash-Next, Thinking OFF — VERIFIED ANSWER GENERATION

This is the only confirmed answer-generation five-workload table in the handoff. It has no matched answer-mode columns for the other two configurations.

| Workload | Output tokens | TTFT | Decode | Completion time | Recorded acceptance length, reported limit 4 |
|---|---|---|---|---|---|
| Chinese business analysis | 671 | 191 ms | 14.40 tok/s | 46.8 s | 1.28 |
| English prose | 1,102 | 185 ms | 25.89 | 42.7 s | 2.28 |
| Python code | 2,537 | 175 ms | 45.69 | 55.7 s | 3.64 |
| SQL generation | 1,335 | 686 ms | 36.38 | 37.4 s | 3.03 |
| Tool calling | 124 | 569 ms | 55.26 | 2.8 s | 3.73 |

These are generation-test completion times, not complete application latency including retrieval, tools, and human-visible rendering. Different output lengths prevent a meaningful workload speed ranking. No Mia thinking-OFF control exists.

## 5. Small Functional Smoke Checks — VERIFIED

| Check | 27B + DFlash2 | Mia Flash-Next | RadixArk Flash-Next |
|---|---|---|---|
| Fixed Tool Calling suite | 9/9 | 9/9 | 9/9 |
| Fixed SQL suite, thinking OFF | 10/10 | 10/10 | 10/10 |
| Difficult SQL, thinking ON, 12k-token budget | No final SQL | No final SQL | Not tested |

Passing fixed checks is not general accuracy, safe enterprise integration, or quality equivalence. The thinking-ON failures demonstrate that generation tokens can be consumed without completing the requested task.

## 6. Engineering-record Observations — PARTIALLY VERIFIED

| Observation | Archived value | Limit |
|---|---|---|
| Mia Flash-Next deployment checks | English 38.4 / Chinese 19.8 tok/s; 600-token tests repeated three times | No original result artifact in this evidence class; output phase not established |
| Mia Flash-Next prefill | About 1,950–2,140 tok/s over 6k–94k inputs | Engineering record only; not comparable to TTFT tables |
| 35B-A3B NVFP4 short test | 102.8–123.4 tok/s; TTFT about 93 ms; 256 thinking tokens | Not a real-task answer-speed or quality result |
| Same SGLang server, adapter-bearing 27B | No speculation about 11–12; borrowed DFlash2 about 21.4 tok/s | Partially supported same-server observation; not a 12-to-33–50 claim |
| Same target with built-in NEXTN | 8.45 tok/s; recorded accept 1.00 | Do not interpret 1.00 as 100% acceptance; the audit reports no useful speculative gain |
| SGLang upgrade, same three questions | 0.5.20: 21.37; 0.5.21: 21.07 tok/s | No demonstrated upgrade speedup; no statistical equivalence claim |

## 7. Invalid Synthetic-token Result

A random-token-ID serving test used input length 8,192 and output length 1,024. The archive recorded 47.27 tok/s and cumulative acceptance length 6.42, versus approximately 3.9–4.9 on real-text tests. The audit identified repetitive degenerate generation. **This is an invalid application-performance benchmark**, retained only as a measurement lesson, not a performance highlight.

## Missing Controls and Reproducibility

Not tested: concurrency above one, multi-user load, prefix-cache reuse, long-conversation soak tests, broad accuracy benchmarks, large-sample BF16/NVFP4 quality, an automatic-KV-dtype control, and formal application latency statistics.

Also missing from this public handoff: complete per-table execution dates, public model revisions, all input lengths, full request reconstruction, repeated-sample confidence estimates, and uniform cold/warm-cache controls. Record these before making stronger comparisons; this is a future measurement plan, not an executed test suite.

No original scripts, prompts, model responses, raw JSON, private evidence indexes, or deployment settings are included. The documents alone are not an executable reproduction package. The original auditor's evidence status is preserved without claiming this documentation pass reverified the original artifacts.

See the [Engineering Journey](ENGINEERING_JOURNEY.md) and [project overview](../README.md).
