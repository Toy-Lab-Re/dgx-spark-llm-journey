# Engineering Journey — Local LLM Inference on GB10

## Scope and Evidence Status

This document follows the public-safe handoff audited by the original development agent on 2026-10-12. **VERIFIED** means that audit identified direct original evidence; **PARTIALLY VERIFIED** means it relied on an engineering record or had missing controls. This portfolio preparation did not re-inspect original logs, rerun benchmarks, or connect to GB10.

The deployment inventory reports six public target checkpoints and ten launched Model × Runtime configurations. A draft checkpoint is an auxiliary model, not an additional target in the six-checkpoint inventory. The six public targets span the first eight rows below; the two later derived-model runtime lanes are part of the deployment evolution, not new public base checkpoints.

## Model and Runtime Inventory

| Target / export | Precision | Runtime and speculation | Execution state and decision |
|---|---|---|---|
| Qwen/Qwen3.8-27B | BF16 | vLLM 0.27.1, no speculation | Launched and measured; retained as a small-test reference |
| nvidia/Qwen3.8-27B-NVFP4 | Mixed NVFP4 / FP8 | vLLM 0.27.1, FP8 KV | Launched and measured; suspended as the daily lane |
| Mia-AiLab/Qwen3.8-Flash-Next-NVFP4 | NVFP4 | Community vLLM build, MTP 3 | Launched and measured; previously used, later replaced |
| nvidia/Qwen3.8-27B-NVFP4 | Mixed NVFP4 / FP8 | SGLang 0.5.20, DFlash2 | Launched and measured; selected daily inference lane |
| RadixArk/Qwen3.8-Flash-Next-NVFP4 | NVFP4, BF16 MTP head | SGLang development build, NEXTN, PLE offload | Launched and measured; retained but not selected |
| Qwen/Qwen3.8-27B | BF16 | SGLang 0.5.20, DFlash2 | Launched; smoke only, no formal benchmark |
| RadixArk/Qwen3.8-27B-NVFP4-BF16-LMHead | NVFP4, BF16 output head | SGLang 0.5.20, DFlash2 | Launched; smoke only, no formal benchmark |
| nvidia/Qwen3.6-35B-A3B-NVFP4 | NVFP4, Marlin MoE backend | vLLM 0.27.1, MTP 3 | Launched; short thinking test, not adopted |
| Derived domain target, earlier runtime lane | Adapter / later merged mixed precision | SGLang 0.5.20, DFlash2 | Launched; referenced only for serving decisions |
| Derived domain target, later runtime lane | Merged and re-quantized mixed precision | SGLang 0.5.21, aligned DFlash2 | Launched candidate; no Production acceptance |

The two derived-model rows summarize runtime evolution, not a claim that every target/export combination was independently benchmarked. Public model revisions and exact community build identifiers are not supplied in this handoff, so none are invented here. Deployment version-lock records were checked by the original auditor but are not published.

Qwen-Image 2.1 with ComfyUI was a separate image-generation smoke exploration and is excluded from the LLM counts. A BF16 MoE training exploration did not establish serving results. EAGLE3 remained literature research; TensorRT-LLM and TensorFold evaluation records were not found. DeepSeek, GLM, Nemotron, a second GB10, TP2, and Mac Studio were not tested.

## 1. Slow BF16 Decode and Misleading First-Token Impressions — VERIFIED

- **Problem:** Interactive requests appeared to wait around 30 seconds before useful output.
- **Hypothesis:** Prefill alone was the bottleneck.
- **Investigation:** The audit found a same-script prompt-length comparison: BF16 TTFT was 0.41 s for 20 tokens and 6.03 s for 6,528 tokens; long-input and reasoning work must be separated from decode.
- **Decision:** Measure TTFT and decode independently, then evaluate the NVFP4 configuration.
- **Result:** BF16 answer decode was 4.40–4.52 tok/s under the recorded thinking-OFF test.
- **Limitation:** Time until a visible answer can include reasoning. A user-perceived wait is not automatically TTFT; no formal end-to-end latency distribution was established.

## 2. NVFP4 as a Complete Configuration Change — VERIFIED

- **Problem:** BF16 consumed substantial memory and decoded slowly.
- **Hypothesis:** A lower-precision serving configuration could improve the resource budget and response rate.
- **Investigation:** Same vLLM and script recorded BF16 4.40–4.52 versus NVFP4 12.22–12.40 tok/s; reported weight-related memory was 55.31 versus 23.47 GiB, with non-torch memory included in the latter.
- **Decision:** Test the full NVFP4 configuration while preserving a BF16 reference.
- **Result:** The recorded configuration improved throughput and long-prompt TTFT. Small smoke checks matched on 7/8 fixed answers and 2/2 needle probes.
- **Limitation:** KV dtype and Tool Parser changed too. Memory accounting is not perfectly identical; neither pure quantization causality nor general quality equivalence follows.

## 3. Select SGLang and DFlash2 Without Inventing a Causal Speedup — VERIFIED / PARTIALLY VERIFIED

- **Problem:** The vLLM 27B lane remained unsuitable for the desired interaction experience.
- **Hypothesis:** A runtime with a compatible draft could improve practical responsiveness.
- **Investigation:** Real-text SGLang/DFlash2 tests have verified numeric outputs, but no retained response text. A cross-model suite recorded TTFT and functional smoke checks. A same-server no-speculation versus DFlash2 observation exists only as a partially verified engineering record.
- **Decision:** Adopt the 27B SGLang/DFlash2 lane using the upstream recipe; preserve alternatives and version locks.
- **Result:** The archived selection was supported by single-stream observations and small functional checks. Later serving aligned drafts with the derived target and moved away from runtime adapters.
- **Limitation:** This does not isolate DFlash2 from runtime, target, precision, or workload changes. Unified memory needs headroom; the archive's reported available-memory observations vary by configuration. The higher-memory-risk threshold described upstream was not reproduced locally.

## 4. A Workload Label Did Not Mean an Answer Was Generated — VERIFIED

- **Problem:** Five workload categories were initially interpreted as application-answer performance.
- **Hypothesis:** Labels such as SQL generation described the measured output.
- **Investigation:** Retained responses for the first four 27B and Mia Flash-Next rows had zero answer characters: the 3,000-token budget was spent on reasoning. Synthetic random-token tests also produced repetitive output and inflated acceptance measurements.
- **Decision:** Distinguish reasoning-only, answer-only, and unknown-phase tests; reject the synthetic result as an application-performance measure.
- **Result:** The RadixArk Flash-Next thinking-OFF table provides confirmed answer-generation observations. It has no matched answer-mode columns for other models.
- **Limitation:** There is no supported cross-model answer-speed ranking. Raw prompts and responses remain private; single-stream measurements do not establish concurrency.

## 5. Flash-Next Export and Runtime Trade-offs — VERIFIED; CAUSE PARTIALLY VERIFIED

- **Problem:** A different Flash-Next export offered a larger context configuration, cache features, and a different PLE memory strategy.
- **Hypothesis:** Better acceptance and caching could make it a preferable lane.
- **Investigation:** The original audit found recipe-porting and file-verification records. Under the same thinking-ON script, the Mia lane had higher decode throughput across five categories, even where the RadixArk lane had better acceptance.
- **Decision:** Retain the alternative without adopting it; port runtime components without allowing an installer to take over Host management.
- **Result:** The alternative launched and its thinking-OFF output was measured. Acceptance alone did not determine throughput.
- **Limitation:** Checkpoint/export, runtime, and offload all changed. NVMe PLE access was a suspected bottleneck, not a proven cause. Cache reuse, long-context advantage, and concurrency were not tested.

## 6. Parser and Streaming Contract Failures — VERIFIED / PARTIALLY VERIFIED

- **Problem:** Tools appeared as text, answers appeared in the wrong response field, or the UI showed delayed chunks and did not finish cleanly.
- **Hypothesis:** The visible symptom came from model behavior alone.
- **Investigation:** The audit distinguishes reasoning-parser evidence from partially supported tool-parser and streaming-engineering accounts. Response-field separation and connection handling explained specific serving symptoms.
- **Decision:** Set model-appropriate reasoning/tool parsers and validate real requests; correct buffered streaming and response termination in the serving adapter.
- **Result:** The reasoning-parser correction is verified in the handoff. Earlier streaming fixes are partially verified. None is a claim of model-quality improvement.
- **Limitation:** Full parser-mode acceptance artifacts are not supplied here. No current timeout, endpoint, interface protection, or deployment details are disclosed.

## Platform and Human/Agent Boundaries

Use system available-memory observations on unified-memory hardware; a missing GPU-memory reading is not proof of unused memory. The recorded policy kept model experiments inside containers, separated Host and model responsibilities, and limited operation to one large model at a time. It does not establish that no Host work ever occurred outside this inference-study scope.

The Project Owner directed requirements, sequence, selection, rollback, and human use review. Claude Code performed most verified model-runtime implementation, scripts, tests, and debugging. Historical Host baseline and monitoring attribution to Codex is partially verified because execution records were unsigned; monitoring/control-panel authorship must not be assigned conclusively to either agent. This Windows session produced documentation only.

本作品呈現的是推理工程案例，不重述 QLoRA 訓練歷程，也不宣稱正式 Production、多機能力或完整資安驗收。內部識別、設定與安全缺口不在公開文件中展開。

See [Benchmark Method](BENCHMARK_METHOD.md) and the [project overview](../README.md).
