# rocBLAS gfx900 Tuning Points (MI25)

Last updated: 2026-03-25
Target: `ROCm-repos_AETS/rocBLAS`

## 1. Scope

This note lists practical tuning points for LLM inference on MI25/gfx900
(ollama + GGML/HIP), ordered by likely impact.

### 1.1 Overlap topics with Tensile note (explicit list)

The following topics currently appear in both notes and are intentionally
tracked as shared context:

1. anchor workload (`gpt-oss` anchor condition)
2. baseline/side lane (`num_batch=512/1024`)
3. `keep_alive` threshold and observability
4. shape observability (top-shape families and lane shifts)
5. runtime path / mixed stack facts
6. direct/indirect dispatch wording and gate status

### 1.2 Role split (authoritative owner)

For maintenance going forward, this rocBLAS note is the primary location for:

- runtime path / mixed stack status (`librocblas.so.5` resolution + path order)
- GEMM shape observation and lane-level shape movement
- client/runtime knobs (`num_thread`, `num_ctx`, `num_predict`, `keep_alive`)
- observability lane operations (baseline/side and stream-window comparison)

The following topics are secondary here and primary in
`../Tensile/Tensile_gfx900-tuning-points.md`:

- fallback asset inventory details
- HSACO extraction pipeline
- disassembly signal summaries
- kernel candidate narrowing process

### 1.3 Wording guard (for README and cross-note consistency)

- `direct dispatch`:
  same-scenario evidence where rocBLAS/Tensile-named dispatch-level traces are visible.
- `indirect link only`:
  fallback + dispatch are both confirmed in the same scenario, but direct
  rocBLAS/Tensile naming is absent.
- `catalog-read evidence` and `dispatch evidence` are different gates and must
  not be treated as interchangeable.

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

## 16. Path recheck + anchor-lane auto-summary (2026-03-25)

Path-resolution recheck (using anchor lane summaries):

- baseline source:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/g4_summary_gpt-oss_latest_20260324_034636.txt`
- side source:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/g4_summary_gpt-oss_latest_20260324_035250.txt`

Observed in both lanes:

- `libggml-hip.so` is loaded from:
  - `/home/limonene/ROCm-project/ollama-src/build-gfx900/lib/ollama/libggml-hip.so`
- `librocblas.so.5` resolves to system ROCm:
  - `/opt/rocm-7.2.0/lib/librocblas.so.5`
- fallback `.dat/.hsaco` assets are read from fork-side Tensile library path:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/build-mi25-gfx900/release/rocblas-install/lib/rocblas/library`

This reconfirms the mixed but intentional runtime stack:

1. GGML HIP backend from `build-gfx900`
2. rocBLAS binary from system ROCm
3. Tensile fallback assets from AETS fork-side path

Automation update:

- Added lane status summarizer:
  - `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-g4-anchor-lanes.sh`
- latest output:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_anchor_lane_status_gpt-oss_latest_20260325_010009.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_anchor_lane_status_gpt-oss_latest_20260325_010009.tsv`

Key result from this aggregate:

- baseline lane: `ok_cases=5`, `direct_hits=5`
- side lane: `ok_cases=3`, `direct_hits=3`
- stream compare rows: all 5 rows kept
  - `direct/fallback/dispatch = 1`
  - `decode_signature_detected` on both lanes

## 17. Non-dot4 candidate shortlist + shape priority (2026-03-25)

Inputs:

- dtype summary:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_gemm_dtype_summary_rocblas_gemm_shapes_g4_rocblas_trace_gpt-oss_latest_20260324_045255_20260324_045658_20260325_010632.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_gemm_dtype_summary_rocblas_gemm_shapes_g4_rocblas_trace_gpt-oss_latest_20260324_045255_20260324_045658_20260325_010632.tsv`
- shape source:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/rocblas_gemm_shapes_g4_rocblas_trace_gpt-oss_latest_20260324_045255_20260324_045658.tsv`

Observed (`gpt-oss` anchor trace, gemm-only rows):

- `total_gemm=501`
- `non_dot4_like=501` (100%)
- `int8_or_i32_like=0`
- dtype mix:
  - `bf16_r|bf16_r|||` -> `288` (57.49%)
  - `f16_r|f16_r|||` -> `144` (28.74%)
  - `f32_r|f32_r|f32_r|f32_r|f32_r` -> `69` (13.77%)

Interpretation:

- [main-node confirmed] The current direct-dispatch anchor is dominated by
  BF16/F16/F32 GEMM signatures, with no int8/i32 signature in gemm rows.
- [inference] For immediate optimization work, non-dot4 paths should be treated
  as the first-class target in this lane.

Priority order for "which shape to stab first":

1. Tier-1 (highest frequency, baseline lane core)
   - `512x512x2880` (19.16%)
   - `2880x512x4096` (9.58%)
   - `4096x512x2880` (9.58%)
2. Tier-2 (decode-tail but still frequent)
   - `512x93x2880` (9.58%)
   - `32x512x2880` (9.18%)
3. Tier-3 (batched-ex family for sensitivity checks)
   - `4608x512x64`, `64x512x4608`, `8192x512x64`, `64x512x8192` (each 4.79%)

Operational implication:

- This closes two pending prep items in the weekly lane:
  1) non-dot4 candidate shortlist
  2) initial shape stab priority
- Next step is per-shape memo split (Tier-1 first), then low-level kernel-side
  checks under the same anchor condition.

## 18. Tier-1 per-shape notes (Queue-A split) (2026-03-25)

Per-shape observation notes were split into dedicated files:

- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/README.md`
- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x512x2880.md`
- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_2880x512x4096.md`
- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4096x512x2880.md`

Captured comparison dimensions in each note:

1. direct dispatch stability
2. gemm-line volume
3. Tensile-like rows
4. prefill/decode proxy difference

Current state:

- Queue-A (Tier-1) split: done.
- Queue-B/C split: done (shape note files added).

## 19. Full-shape prefill/full compare tables (Queue-B/C visibility) (2026-03-25)

Added helper:

- `/home/limonene/ROCm-project/ROCm-MI25-build/compare-rocblas-shape-counts.sh`

Latest outputs:

- baseline lane compare:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_shape_prefill_full_compare_g4_rocblas_trace_gpt-oss_latest_20260325_011553__g4_rocblas_trace_gpt-oss_latest_20260325_011629_20260325_012104.tsv`
- side lane compare:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocblas_shape_prefill_full_compare_g4_rocblas_trace_gpt-oss_latest_20260325_011411__g4_rocblas_trace_gpt-oss_latest_20260325_011439_20260325_012104.tsv`

Observed [main-node confirmed]:

- both lanes: `__TOTAL__ delta = 0` (`prefill_count == full_count`)
- baseline top includes Queue-B/C candidates:
  - `512x93x2880` (48), `32x512x2880` (46)
  - `4608x512x64`, `64x512x4608`, `8192x512x64`, `64x512x8192` (24 each)
- side top includes batch-shifted variants:
  - `32x1024x2880` (69)
  - `5120x1024x64`, `64x1024x5120`, `8192x1024x64`, `64x1024x8192` (36 each)

Interpretation [inference]:

- Queue-B/C are now visible in the same prefill/full compare framework.
- Current proxy split still indicates prefill-dominant signature under this anchor.
- This keeps Queue-B/C ready for deeper per-kernel correlation without changing
  the established gate conditions.

Queue-B/C note files:

- Queue-B:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x93x2880.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_32x512x2880.md`
- Queue-C:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4608x512x64.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x4608.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_8192x512x64.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x8192.md`

## 20. Kernel candidate narrowing from rocprof summaries (2026-03-25)

Added helper:

- `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-kernel-candidates.sh`

Purpose:

- Read prefill/full `rocprofv3_summary_*.txt`.
- Resolve `kernel_trace_file` from each summary.
- Compare kernel-name counts and classify likely matmul-path candidates.

Latest outputs:

- baseline lane:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150.tsv`
- side lane:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011425__rocprofv3_summary_gpt-oss_latest_20260325_011502_20260325_013150.tsv`

Observed [main-node confirmed]:

- baseline dispatch rows: `2384 -> 25204` (`delta=22820`)
- side dispatch rows: `2383 -> 25021` (`delta=22638`)
- both lanes keep the same top matmul-related families:
  - `mul_mat_vec_f<...>`
  - `mul_mat_vec_q<(ggml_type)39, ...>`
  - `mul_mat_q<(ggml_type)39, ...>`
  - `Cijk_*` (Tensile kernel names)

Interpretation [inference]:

- Candidate narrowing is now scriptable/repeatable from summary inputs.
- For next step, `Cijk_*` kernels are suitable starting points for HSACO
  extraction + disassembly target minimization under the current anchor.

## 21. HSACO mapping/extraction for disassembly target set (2026-03-25)

Added helpers:

- `/home/limonene/ROCm-project/ROCm-MI25-build/map-kernel-candidates-to-hsaco.sh`
- `/home/limonene/ROCm-project/ROCm-MI25-build/extract-hsaco-targets.sh`

Inputs:

- baseline map source:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150.tsv`
- side map source:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011425__rocprofv3_summary_gpt-oss_latest_20260325_011502_20260325_013150.tsv`

Observed [main-node confirmed]:

- `Cijk_*` 4 candidates:
  - matched to `*gfx900*.hsaco`: 3
  - unmatched in current gfx900 library scan: 1 (`...ISA900...`)
- extracted target set:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150_20260325_013452_20260325_013541`
  - file count: 3
  - total size: `876K`

Target HSACO files:

- `TensileLibrary_Type_BB_HPA_Contraction_l_Alik_Bljk_Cijk_Dijk_fallback_gfx900.hsaco`
- `TensileLibrary_Type_HH_Contraction_l_Alik_Bljk_Cijk_Dijk_fallback_gfx900.hsaco`
- `TensileLibrary_Type_HS_HPA_Contraction_l_Alik_Bljk_Cijk_Dijk_fallback_gfx900.hsaco`

Interpretation [inference]:

- Disassembly target scope is now reduced to a small, explicit 3-file set.
- This is sufficient to begin instruction-level checks without broad scanning.

## 22. Disassembly signal summary on extracted 3-file set (2026-03-25)

Added helper:

- `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-hsaco-disasm-signals.sh`

Inputs:

- extracted target dir:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150_20260325_013452_20260325_013541`

Outputs:

- summary txt:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_011606__rocprofv3_summary_gpt-oss_latest_20260325_011645_20260325_013150_20260325_013452_20260325_013541_20260325_013821.txt`

Observed [main-node confirmed]:

- `dot4_positive_files=0`
- `mfma_positive_files=0`
- `packed_positive_files=1`
- `memory_positive_files=3`
- packed instruction example:
  - `v_pk_fma_f16 ...` (HH fallback file)
- memory instruction examples:
  - `global_load_dword`, `ds_read2_b32`, `ds_write_b16`

Interpretation [inference]:

- In the current extracted target set, instruction signature is dominated by
  FMA-like + memory operations rather than dot4/mfma.
- For deeper manual disassembly, prioritize:
  1) `Type_HH ... fallback_gfx900.hsaco` (packed-rich)
  2) `Type_BB_HPA ... fallback_gfx900.hsaco`
  3) `Type_HS_HPA ... fallback_gfx900.hsaco`

## 23. Observation-only deepening cycle (2026-03-25 02:23 JST)

Scope:

- No kernel/source modification in this cycle.
- Focus only on deeper observation granularity:
  1) top-shape re-observation
  2) baseline/side stability check
  3) prefill/full proxy + stream-window split comparison
  4) candidate -> hsaco -> disasm note refresh

Anchor lane re-observation [main-node confirmed]:

- baseline (`num_batch=512`):
  - summary: `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022355.txt`
  - `direct/fallback/dispatch=1`, `gemm_lines=1002`, shapes:
    - `512x512x2880=192`
    - `2880x512x4096=96`
    - `4096x512x2880=96`
- side (`num_batch=1024`):
  - summary: `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022435.txt`
  - `direct/fallback/dispatch=1`, `gemm_lines=1336`, shapes:
    - `512x1024x2880=288`
    - `2880x1024x4096=144`
    - `4096x1024x2880=144`

Prefill/full shape-proxy split [main-node confirmed]:

- baseline split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022531.txt`
- side split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022637.txt`
- both lanes:
  - `decode_delta_gemm_lines=0`
  - `decode_delta_target_shape_hits=0`
  - `phase_split_status=prefill_dominant_signature`

Stream phase-window split [main-node confirmed]:

- baseline sweep (`num_predict=64,128,256`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022802.txt`
- side sweep (`num_predict=64,128,256`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022953.txt`
- both lanes:
  - `ok_cases=3`
  - `decode_signature_cases=3`
  - all rows `direct/fallback/dispatch=1`
  - `decode_kernel_tensile_like_rows=167`

Candidate -> HSACO refresh [main-node confirmed]:

- candidates:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022651__rocprofv3_summary_gpt-oss_latest_20260325_022727_20260325_023315.txt`
- map (both lanes):
  - `total_candidates=4`, `matched_candidates=3`, unmatched `...ISA900...` persists
- extract + disasm:
  - extract manifest: `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347.txt`
  - disasm summary: `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347_20260325_023354.txt`
  - signals unchanged: `dot4=0`, `mfma=0`, `packed_positive_files=1`, `memory_positive_files=3`

Interpretation:

- [inference] The anchor remains stable across baseline/side and across the tested
  decode-length range in stream-window probes.
- [inference] Shape-proxy split and stream-window split expose different layers of
  evidence and should be interpreted together, not merged into one claim.
- [inference] This cycle strengthens observation fidelity without requiring low-level
  code changes.

## 24. Tier-1 per-shape kernel-priority memos (2026-03-25)

Added per-shape memo files for observation-first kernel priority review:

- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x512x2880_kernel_priority.md`
- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_2880x512x4096_kernel_priority.md`
- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4096x512x2880_kernel_priority.md`

Characteristics:

- observation-only (no kernel/source modifications)
- fixed evidence chain:
  - shape stability (baseline/side)
  - stream-window decode signature
  - lane-level `Cijk_*` candidate ranking
  - HSACO mapping status (`3 matched + 1 unmatched`)
- Mermaid map included in each memo for fast visual review.

## 25. Queue-B/C per-shape kernel-priority memos (2026-03-25)

Extended the same observation-only memo format to Queue-B/C:

- Queue-B:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_512x93x2880_kernel_priority.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_32x512x2880_kernel_priority.md`
- Queue-C:
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_4608x512x64_kernel_priority.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x4608_kernel_priority.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_8192x512x64_kernel_priority.md`
  - `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_64x512x8192_kernel_priority.md`

Status:

- no low-level code change
- no Tensile asset rewrite
- observation-only chain preserved:
  - shape stability
  - stream-window decode signature
  - lane-level `Cijk_*` ranking
  - HSACO map (`3 matched + 1 unmatched`)

## 26. Cross-queue one-sheet overview (2026-03-25)

Added one-page overview across all 9 shape-priority memos:

- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_priority_overview.md`

Contains:

- Queue map (Tier-1 + Queue-B + Queue-C)
- baseline/side observed-count table
- shared candidate/HSACO layer summary
- Mermaid diagrams for quick cross-check

Role:

- navigation and prioritization only
- no new low-level claim beyond existing evidence

## 27. Decode-signature reproducibility check (2026-03-25 03:31-03:33 JST)

Scope:

- observation-only rerun (no source/kernel edit)
- fixed anchor: `MODEL=gpt-oss:latest`, `ROCBLAS_LAYER=9`
- lane comparison:
  - baseline: `NUM_BATCH=512`
  - side: `NUM_BATCH=1024`

Executed:

- phase-window sweep (num_predict `64,128,256,512,1024`)
  - baseline:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_031811.txt`
  - side:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_032242.txt`
- prefill/full split (`prefill_num_predict=1`, `full_num_predict=128`)
  - baseline:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_032955.txt`
  - side:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_033100.txt`

Observed [main-node confirmed]:

- phase-window sweep:
  - baseline: `ok_cases=5`, `decode_signature_cases=5`, `prefill_dominant_cases=0`
  - side: `ok_cases=5`, `decode_signature_cases=5`, `prefill_dominant_cases=0`
  - all rows (both lanes): `direct_rocblas_or_tensile_dispatch=1`,
    `fallback_confirmed=1`, `dispatch_confirmed=1`,
    `decode_kernel_tensile_like_rows=167`, `prefill_kernel_tensile_like_rows=0`
- prefill/full split:
  - both lanes: `phase_split_status=prefill_dominant_signature`
  - both lanes: `decode_delta_gemm_lines=0`
  - both lanes: `decode_delta_target_shape_hits=0`
  - side lane split target hits remain 0 for baseline-target shape set
    (`512x512x2880`, `2880x512x4096`, `4096x512x2880`)

- side lane target-shape correction rerun:
  - summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_033458.txt`
  - corrected target set:
    - `512x1024x2880`, `2880x1024x4096`, `4096x1024x2880`
  - observed hits:
    - `shape_512_1024_2880=288`
    - `shape_2880_1024_4096=144`
    - `shape_4096_1024_2880=144`
  - `phase_split_status` stays `prefill_dominant_signature`

Interpretation [inference]:

- For the current anchor, decode-side signature is reproducible in the
  stream phase-window lane (`5/5` in both baseline and side).
- The prefill/full proxy split and stream-window split should still be treated
  as separate evidence layers; they are not interchangeable gates.
- For side-lane split comparisons, target-shape set should be lane-aware
  (`*x1024x*` family) when shape-hit deltas are evaluated.

## 28. Low-level entry start (shape-first, no patch yet) (2026-03-25 10 JST)

Scope:

- start low-level optimization from the highest-confidence shape pair
  without touching source kernels yet.
- target pair:
  - baseline: `512x512x2880`
  - side: `512x1024x2880`

Executed [main-node confirmed]:

- refresh candidate extraction from latest split runs:
  - baseline:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092328__rocprofv3_summary_gpt-oss_latest_20260325_092357_20260325_105550.txt`
  - side:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092433__rocprofv3_summary_gpt-oss_latest_20260325_092510_20260325_105556.txt`
- refreshed candidate -> hsaco mapping (both lanes):
  - baseline map:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092328__rocprofv3_summary_gpt-oss_latest_20260325_092357_20260325_105550_20260325_105606.txt`
  - side map:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092433__rocprofv3_summary_gpt-oss_latest_20260325_092510_20260325_105556_20260325_105606.txt`
- lane automation wrapper added:
  - `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-k1-entry.sh`
  - latest output:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/k1_entry_lane_check_20260325_110634.txt`

Observed [main-node confirmed]:

- both lanes: `total_candidates=4`, `matched_candidates=3`
- unmatched candidate remains `..._SB_..._ISA900...`

Interpretation [inference]:

- Entry gate is now satisfied for shape-first low-level work.
- First touchpoint remains `K1 (BBS_BH)` with `K2/K3` as secondary while
  keeping `K4` as unmatched watchpoint.
- This section marks "entry started"; source/kernel edits are still pending.

## 29. K1 entry gate hardening (2026-03-25 11 JST)

Scope:

- keep observation-first flow while tightening "first touchpoint" confidence.
- no rocBLAS source edit in this step.

Executed [main-node confirmed]:

- lane wrapper:
  - `/home/limonene/ROCm-project/ROCm-MI25-build/summarize-k1-entry.sh`
  - output:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/k1_entry_lane_check_20260325_110634.txt`
- HSACO/disasm cross-check for baseline/side maps:
  - baseline disasm summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092328__rocprofv3_summary_gpt-oss_latest_20260325_092357_20260325_110634_20260325_110635_20260325_110812_20260325_110821.txt`
  - side disasm summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_092433__rocprofv3_summary_gpt-oss_latest_20260325_092510_20260325_110636_20260325_110636_20260325_110812_20260325_110822.txt`

Observed [main-node confirmed]:

- baseline/side both:
  - `total_candidates=4`, `matched_candidates=3`
  - `K1_full=96`, `K1_match=1`
  - `K4_match=0`
- disasm aggregate also matches between lanes:
  - `dot4_positive_files=0`, `mfma_positive_files=0`
  - `packed_positive_files=1`, `memory_positive_files=3`

Interpretation [inference]:

- K1-first entry gate is now stable across baseline/side at
  candidate -> hsaco -> disasm levels.
- Next low-level action should stay narrowly scoped to K1-first A/B verification.

## 30. Runtime-path A/B check at K1 entry (2026-03-25 14 JST)

Scope:

- observation-only A/B check with fixed anchor
  (`MODEL=gpt-oss:latest`, `NUM_BATCH=512`, `NUM_CTX=8192`,
  `NUM_PREDICT=128`, `ROCBLAS_LAYER=9`).
- isolate runtime path by changing only `ROCBLAS_TENSILE_LIBPATH`.
- no rocBLAS source patch in this step.

Lane setup [main-node confirmed]:

- AETS lane:
  - `ROCBLAS_TENSILE_LIBPATH=/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/build-mi25-gfx900/release/rocblas-install/lib/rocblas/library`
  - link summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_link_summary_gpt-oss_latest_20260325_135852.txt`
- system lane:
  - `ROCBLAS_TENSILE_LIBPATH=/opt/rocm-7.2.0/lib/rocblas/library`
  - strace raw prefix:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/g4_strace_openat_gpt-oss_latest_20260325_140141.log*`
  - rocprof summary:
    - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/rocprofv3_summary_gpt-oss_latest_20260325_140345.txt`
- A/B table:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_runtime_path_ab_compare_20260325_140536.tsv`

Observed [main-node confirmed]:

- AETS lane:
  - `fallback_confirmed=1`, `dispatch_confirmed=1`,
    `direct_rocblas_or_tensile_dispatch=1`
  - `rocblas_trace_gemm_lines=1002`
  - `kernel_dispatch_rows=21664`, `kernel_tensile_like_rows=167`
- system lane:
  - `fallback_confirmed=0`, `dispatch_confirmed=0`,
    `direct_rocblas_or_tensile_dispatch=0`
  - `rocblas_trace_gemm_lines=0`
  - `kernel_dispatch_rows=0`, `kernel_tensile_like_rows=0`
  - strace confirms `librocblas.so.5` resolved from
    `/opt/rocm-7.2.0/lib/librocblas.so.5`

Tooling note:

- Current `g4-fallback-strace-check.sh` exits early when fallback matches are
  zero (no `.dat/.hsaco` lines), so system-lane fallback counts were taken from
  raw strace files directly.

Interpretation [inference]:

- Under fixed anchor conditions, runtime path is a high-impact observability
  knob at K1 entry.
- This remains path-level evidence only; no kernel-level causal mapping is
  claimed in this section.
