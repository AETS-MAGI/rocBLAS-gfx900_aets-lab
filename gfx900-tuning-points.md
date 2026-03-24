# rocBLAS gfx900 Tuning Points (MI25)

Last updated: 2026-03-24
Target: `ROCm-repos_AETS/rocBLAS`

## 1. Scope

This note lists practical tuning points for LLM inference on MI25/gfx900
(ollama + GGML/HIP), ordered by likely impact.

Current facts:

- [main-node confirmed] `librocblas.so.5` may resolve from system ROCm.
- [main-node confirmed] Tensile fallback assets are read from fork-side paths.
- [inference] We must separate three layers when analyzing changes:
  1) rocBLAS binary, 2) Tensile assets, 3) runtime path order.

## 2. Priority order

### P0: Freeze runtime path resolution

Goal:

- Make benchmark results reproducible.

Checkpoints:

- Which `librocblas.so.5` is actually loaded at runtime.
- Path order for `ROCBLAS_TENSILE_LIBPATH`, `LD_LIBRARY_PATH`, `OLLAMA_LIBRARY_PATH`.

Evidence:

- `strace` (`openat/openat2`)
- `vega_path_check_logs` summaries

### P1: Type-level fallback asset visibility (with Tensile)

Goal:

- Quantify which fallback catalogs are read by type (for example `Type_HH`).

Checkpoints:

- Type counts for `TensileLibrary_Type_*_fallback.dat/.hsaco`
- Whether type distribution changes by model/config

Note:

- This is catalog-read evidence, not dispatch evidence.

### P2: Connect client/runtime knobs to effective math path

Goal:

- Map client knobs (`num_thread`, `keep_alive`, preset) to rocBLAS/Tensile behavior.

Current baseline:

- `preset=gfx900_safe`
- `keep_alive=10m` is consistently better on tinyllama and qwen2.5:7b.
- Cross-model default: `num_thread=4`
- tinyllama throughput-only profile: `num_thread=6`
- tinyllama shows a small `balanced` advantage vs `safe`, but research baseline stays `safe`.

### P3: Gate before modifying rocBLAS source

Do not modify rocBLAS core paths until:

1. P0/P1 evidence is stable.
2. `balanced/longctx` reproducibility is measured at n>=10.
3. At least one dispatch-level trace is captured.

## 3. Execution template

```bash
cd /home/limonene/ROCm-project/multi_llm-client
scripts/phase3_bench.sh thread-sweep --repeat 10 --preset gfx900_safe --threads 2,4,6 --prompt "short test"
scripts/phase3_bench.sh keepalive-sweep --repeat 10 --preset gfx900_safe --keep-alive-values 0s,10m --model tinyllama:latest --prompt "short test"
```

## 4. Current decision

- Stay in evidence-first mode.
- Prioritize runtime path fixing and type-level observations before low-level code changes.

## 5. New instrumentation (2026-03-24)

Added on the canonical main-node workflow side (`ROCm-MI25-build`):

- `g4-fallback-strace-check.sh`
  - now supports `STRACE_TIMESTAMP=1` (default) for `strace -tt`
  - now supports `PROBE_ROCBLAS_LOG=1` to capture rocBLAS trace files
- `summarize-fallback-phases.sh`
  - summarizes fallback `.dat` / `.hsaco` timing spans per pid log

Latest probe snapshot:

- summary: `g4_summary_tinyllama_latest_20260324_014707.txt`
- `rocblas_trace_lines=1`
- `rocblas_trace_handle_lines=1`
- `rocblas_trace_gemm_lines=0`
- same result under `ROCBLAS_LAYER=63`:
  - `g4_summary_tinyllama_latest_20260324_015056.txt`
  - `rocblas_trace_gemm_lines=0`

Interpretation:

- We can now confirm rocBLAS trace activation path.
- Dispatch-level GEMM evidence is still missing from current trace output.
- Next step is trace-granularity escalation before changing rocBLAS kernels.

Update from rocprofv3 kernel trace:

- A separate `rocprofv3` probe now confirms dispatch-level kernel traces are collectible
  (`kernel_dispatch_rows=3605` on tinyllama short run).
- However, observed dispatch names are still ggml-hip-side kernels (`mul_mat_q`, etc.),
  not explicit rocBLAS/Tensile-named kernels.
- Therefore, for rocBLAS tuning decisions, we still need one run linking:
  fallback asset access + rocBLAS/Tensile dispatch evidence.

Integrated link status update:

- `g4-fallback-dispatch-link-check.sh` now orchestrates both probes under the same condition.
- Latest tinyllama result:
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `direct_rocblas_or_tensile_dispatch=0`
  - `link_status=indirect_link_only_same_scenario`
- Latest qwen2.5:7b result:
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `direct_rocblas_or_tensile_dispatch=0`
  - `link_status=indirect_link_only_same_scenario`
- This closes the "separate-run evidence" gap on two models, but direct rocBLAS/Tensile dispatch naming is still open.

ROCBLAS_LAYER visibility sweep update:

- Added `ROCm-MI25-build/g4-rocblas-layer-sweep.sh` and ran `1,8,9,15,63`
  on `tinyllama` and `qwen2.5:7b`.
- Common result:
  - layer `8` only: no trace lines
  - layer `1/9/15/63`: only `rocblas_create_handle` (no GEMM/internal backend lines)
- Operational default is now fixed to `ROCBLAS_LAYER=9` (trace + internal).
- Conclusion:
  - Layer tuning itself is no longer the blocker.
  - Next blocker is getting a workload path that emits rocBLAS GEMM-level logs.

Workload-path breakthrough update:

- Ran higher-density workload sweep (`qwen2.5:7b` / `deepseek-r1:14b`,
  `NUM_PREDICT=512`, `prompt_profile=long|math|code`):
  - `g4_workload_path_sweep_20260324_023631.txt`
  - `direct_hits=0`
- Then ran `gpt-oss:latest` single-case probe:
  - `g4_link_summary_gpt-oss_latest_20260324_024249.txt`
  - `direct_rocblas_or_tensile_dispatch=1`
  - `rocblas_trace_gemm_lines=1002`
  - `kernel_tensile_like_rows=167`
- This is the first confirmed same-scenario direct link between:
  1) fallback asset access and
  2) rocBLAS/Tensile dispatch evidence.

New shape-level extraction:

- Added `ROCm-MI25-build/summarize-rocblas-gemm-shapes.sh`.
- On `gpt-oss` trace (`g4_rocblas_trace_gpt-oss_latest_20260324_024249.log`):
  - `gemm_api_lines=501`
  - `internal_tensile_lines=501`
  - dominant shapes include:
    - `512x512x2880` (`rocblas_gemm_ex` + `rocblas_gemm_tensile_backend`)
    - `4096x512x64` / `64x512x4096` (`rocblas_gemm_batched_ex`)
    - `2880x512x4096`, `4096x512x2880` (`rocblas_gemm_ex`)

Updated immediate focus:

- Keep `ROCBLAS_LAYER=9` as observability default.
- Use `gpt-oss:latest` as the direct-dispatch anchor workload.
- Prioritize tuning/investigation around the extracted high-frequency shapes
  before broad source-level edits.

Formal reflection (fact / interpretation / implication):

1. Fact
   - Under `gpt-oss:latest`, both `rocblas_gemm_ex` and
     `rocblas_gemm_tensile_backend` were observed repeatedly.
   - `direct_rocblas_or_tensile_dispatch=1` is confirmed.
2. Interpretation
   - Direct dispatch names were absent for tinyllama/qwen in earlier runs but
     appeared with a different workload.
   - So the main blocker was likely workload/path conditions, not `ROCBLAS_LAYER`.
3. Implication
   - We now have a stable probe condition that exposes direct rocBLAS/Tensile dispatch.
   - Use this as the baseline to compare model/precision/path deltas.

## 6. Anchor-shape sweep entrypoint (2026-03-24)

Runtime-side probe scripts in `ROCm-MI25-build` were extended so we can sweep
runtime knobs without changing low-level code:

- `NUM_CTX`
- `NUM_BATCH`
- `NUM_THREAD`
- `KEEP_ALIVE`

New orchestrator:

- `ROCm-MI25-build/g4-gptoss-anchor-shape-sweep.sh`

What it adds:

- fixed anchor defaults: `MODEL=gpt-oss:latest`, `ROCBLAS_LAYER=9`
- case matrix execution via `g4-fallback-dispatch-link-check.sh`
- per-case shape counters for first-priority targets:
  - `512x512x2880`
  - `4096x512x64`
  - `64x512x4096`
  - `2880x512x4096`
  - `4096x512x2880`

Why this matters for rocBLAS-side tuning:

- We can now compare whether runtime knobs change direct dispatch visibility and
  shape frequency before touching kernels.
- This keeps the current phase evidence-first and avoids conflating runtime-path
  changes with low-level source edits.

Validation snapshot (main-node, 2026-03-24):

- run:
  - `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=8192 NUM_BATCH_LIST=512 KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_033556.txt`
- key metrics:
  - `direct_hits=1`
  - `rocblas_trace_gemm_lines=1002`
  - target hits:
    - `512x512x2880=192`
    - `2880x512x4096=96`
    - `4096x512x2880=96`

Additional batch comparison (`num_batch=512,1024`):

- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_033756.txt`
- both cases kept `direct_rocblas_or_tensile_dispatch=1`
- with `num_batch=1024`, dominant shapes shifted to:
  - `512x1024x2880`
  - `2880x1024x4096`
  - `4096x1024x2880`
- implication:
  - shape-target lists should be conditioned on runtime batch context.

## 7. Canonical anchor freeze (baseline vs side)

Canonical profile reference:

- `ROCm-MI25-build/ROCm-MI25-tips/G4_gptoss_anchor_profile.md`

Baseline lane (default for tuning comparisons):

- `MODEL=gpt-oss:latest`
- `ROCBLAS_LAYER=9`
- `NUM_CTX=8192`
- `NUM_BATCH=512`
- `NUM_PREDICT={64,128,256}`
- summary: `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_034636.txt`
- stable outcomes across all 3 predict values:
  - `direct_hits=3/3`
  - `rocblas_trace_gemm_lines=1002`
  - dominant `*x512x*` shapes remain unchanged

Side lane (shape-shift sensitivity):

- `NUM_BATCH=1024` with 1024-target shapes
- summary: `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_035250.txt`
- stable outcomes across all 3 predict values:
  - `direct_hits=3/3`
  - `rocblas_trace_gemm_lines=1336`
  - dominant shapes move to `*x1024x*`

Operational rule:

- Use baseline lane for primary before/after tuning judgments.
- Use side lane to validate whether changes are robust under batch-driven shape migration.

## 8. Single-knob sweep result: `num_ctx` under baseline512

Run:

- `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=4096,6144,8192,12288 NUM_BATCH_LIST=512 TARGET_SHAPES='512x512x2880,2880x512x4096,4096x512x2880' KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_040223.txt`

Observed (all 4 ctx cases):

- `direct_rocblas_or_tensile_dispatch=1`
- `rocblas_trace_gemm_lines=1002`
- shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested range, `num_ctx` is not a primary lever for rocBLAS shape-frequency
  movement under baseline512.
- Next knobs should prioritize `num_thread`, `keep_alive`, prompt profile, and `num_predict`.

## 9. Single-knob sweep result: `num_thread` under baseline512

Run:

- `MODEL=gpt-oss:latest NUM_PREDICT_LIST=128 NUM_CTX_LIST=8192 NUM_BATCH_LIST=512 NUM_THREAD_LIST=2,4,6,8 TARGET_SHAPES='512x512x2880,2880x512x4096,4096x512x2880' KEEP_ALIVE_LIST=5m RUNS_PER_CASE=1 ./g4-gptoss-anchor-shape-sweep.sh`
- summary:
  - `g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260324_040941.txt`

Observed (all 4 thread cases):

- `direct_rocblas_or_tensile_dispatch=1`
- `rocblas_trace_gemm_lines=1002`
- shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested range, `num_thread` is also not a primary lever for shape-frequency
  movement under baseline512.
- Next knobs should shift to prompt profile and extended `num_predict` ranges.

## 10. Single-knob sweep result: prompt profile under baseline512

Run setup:

- baseline fixed: `MODEL=gpt-oss:latest`, `NUM_PREDICT=128`, `NUM_CTX=8192`,
  `NUM_BATCH=512`, `KEEP_ALIVE=5m`
- profiles: `short`, `long`, `code`, `math`
- compare note:
  - `g4_baseline512_prompt_profile_sweep_compare_20260324_042420.txt`

Observed (all 4 profiles):

- `direct_rocblas_or_tensile_dispatch=1`
- `rocblas_trace_gemm_lines=1002`
- shape hits unchanged:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`

Implication:

- In this tested profile set, prompt style/length did not move the baseline512
  rocBLAS shape-frequency observation.
- The next single-knob target should be extended `num_predict` ranges.

## 11. Single-knob sweep result: extended `num_predict` under baseline512

Run setup:

- baseline fixed: `MODEL=gpt-oss:latest`, `NUM_CTX=8192`,
  `NUM_BATCH=512`, `KEEP_ALIVE=5m`
- `NUM_PREDICT={64,128,256,512,1024}`
- compare note:
  - `g4_baseline512_numpredict_sweep_compare_20260324_043625.txt`

Observed:

- `direct_rocblas_or_tensile_dispatch=1` for all 5 cases
- `rocblas_trace_gemm_lines=1002` for all 5 cases
- top-3 shape hits unchanged for all 5 cases:
  - `512x512x2880=192`
  - `2880x512x4096=96`
  - `4096x512x2880=96`
- generation-side evidence still scales:
  - `eval_count` tracks requested length (`64/128/256/512`)
  - `1024` case ended at `eval_count=797` with `done_reason=stop`

Implication:

- Under baseline512, extending decode length did not change the observed rocBLAS
  GEMM signature in current trace mode.
- This suggests the currently observed signature is likely prefill-dominant.
- Next step should split prefill vs decode observation windows.

## 12. Stream phase-window sweep (`num_predict=64..1024`) under baseline512

Run setup (main-node, 2026-03-24):

- command:
  - `NUM_PREDICT_LIST=64,128,256,512,1024 ./g4-stream-phase-window-sweep.sh`
- summary:
  - `ROCm-MI25-build/vega_path_check_logs/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.txt`
- table:
  - `ROCm-MI25-build/vega_path_check_logs/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.tsv`

Observed:

- all 5 cases succeeded (`ok_cases=5`, `failed_cases=0`)
- all 5 cases preserved gate signals:
  - `direct_rocblas_or_tensile_dispatch=1`
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
- all 5 cases showed:
  - `phase_split_status_proxy=decode_signature_detected`
  - `prefill_kernel_tensile_like_rows=0`
  - `decode_kernel_tensile_like_rows=167`
  - `stream_first_token_channel=thinking`

Formal reflection (fact / interpretation / implication):

1. Fact
   - Under baseline512 + gpt-oss anchor, stream-window probes stayed
     `decode_signature_detected` across `num_predict=64..1024`.
   - Direct rocBLAS/Tensile dispatch evidence remained present in every case.
2. Interpretation
   - Increasing decode length changes total runtime but does not destabilize the
     dispatch-observable anchor condition.
   - The "first token via thinking channel" behavior is stable for this model path.
3. Implication
   - We can treat this stream-window sweep profile as a robust observability lane
     for further rocBLAS/Tensile shape-level comparisons.
   - Remaining caution: current split is still proxy-based, not strict token-level attribution.

## 13. Baseline512 vs side1024 in stream phase-window lane

Comparison inputs (main-node, 2026-03-24):

- baseline tsv:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_105527.tsv`
- side1024 tsv:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260324_122317.tsv`
- derived compare table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_batch_compare_gpt-oss_latest_20260324_123206.tsv`

Observed:

- both lanes kept, for all `num_predict={64,128,256,512,1024}`:
  - `direct_rocblas_or_tensile_dispatch=1`
  - `fallback_confirmed=1`
  - `dispatch_confirmed=1`
  - `phase_split_status_proxy=decode_signature_detected`
  - `decode_kernel_tensile_like_rows=167`
- side1024 increased total stream wall time in every case
  while preserving the same observability signature.

Formal reflection (fact / interpretation / implication):

1. Fact
   - Raising `num_batch` from 512 to 1024 did not change dispatch gate outcomes
     or phase-window proxy class in this lane.
   - It consistently increased total wall-time.
2. Interpretation
   - In the current setup, batch acts primarily as a runtime-cost scaler,
     not as a selector for a different stream-visible dispatch signature.
3. Implication
   - Keep baseline512 as the canonical tuning judgment lane.
   - Keep side1024 as a sensitivity lane for runtime scaling and robustness checks.

## 14. Stream observability sensitivity: `keep_alive` threshold

Runs (main-node, baseline512, `num_predict=128`):

- sweep A (`0s,5m,30m`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_123600.tsv`
- 0s recheck (2 runs):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_keepalive_0s_recheck_20260324_123825.tsv`
- sweep B (`1s,10s,30s,5m`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_123938.tsv`

Observed:

- `keep_alive=0s` and `1s` repeatedly showed:
  - `dispatch_confirmed=0`
  - `phase_split_status_proxy=unavailable`
  - rocprof summary with `trace_file_count=0`, `csv_file_count=0`
- `keep_alive=10s/30s/5m` consistently showed:
  - `dispatch_confirmed=1`
  - `phase_split_status_proxy=decode_signature_detected`
  - `decode_kernel_tensile_like_rows=167`

Formal reflection (fact / interpretation / implication):

1. Fact
   - Very short keep-alive values (`0s`, `1s`) caused reproducible loss of
     rocprof dispatch evidence in this stream observability lane.
2. Interpretation
   - This is an observability-window issue, not a direct fallback/path failure:
     direct/fallback gates can still be positive while rocprof phase split becomes unavailable.
3. Implication
   - For stable rocBLAS/Tensile stream-phase evidence collection, set
     `keep_alive>=10s` as an operational minimum.

## 15. Cross-batch confirmation of `keep_alive>=10s`

Additional check (main-node, `num_predict=128`):

- side1024 (`num_batch=1024`) keep-alive sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_sweep_gpt-oss_latest_20260324_124412.tsv`
- baseline/side combined table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_keepalive_batch_compare_gpt-oss_latest_20260324_124713.tsv`

Observed:

- same threshold pattern across both lanes:
  - `keep_alive=1s` -> `dispatch_confirmed=0`, `phase_split_status_proxy=unavailable`
  - `keep_alive>=10s` -> `dispatch_confirmed=1`, `decode_signature_detected`

Implication:

- The minimum keep-alive requirement is not specific to baseline512.
- Apply `keep_alive>=10s` uniformly when collecting stream-phase rocBLAS evidence.
