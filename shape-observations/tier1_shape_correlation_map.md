# Tier-1 Shape Correlation Map (Observation-Only)

Last updated: 2026-03-25
Scope: `gpt-oss` anchor lane (`ROCBLAS_LAYER=9`, baseline/side), no kernel/source modification.

## 1. What This Map Is

This note visualizes the current evidence chain:

1. top shapes are stable in baseline/side lanes
2. prefill/full and stream-window probes are both captured
3. candidate kernels are narrowed
4. candidates are mapped to fallback HSACO files
5. disasm signal summaries are attached

This is an observation map, not a code-change plan.

## 2. Evidence Pipeline

```mermaid
flowchart LR
    A[gpt-oss Anchor<br/>baseline=512 / side=1024] --> B[Top Shapes<br/>512x512x2880<br/>2880x512x4096<br/>4096x512x2880]
    B --> C1[Prefill/Full Shape Proxy<br/>phase_split_status=prefill_dominant_signature]
    B --> C2[Stream Phase Window<br/>decode_signature_detected]
    C1 --> D[Kernel Candidates<br/>Cijk_* x4]
    C2 --> D
    D --> E[HSACO Map<br/>matched=3 / unmatched=1]
    E --> F[Disasm Signals<br/>dot4=0 mfma=0<br/>packed_positive=1 memory_positive=3]
```

## 3. Tier-1 Shape View

```mermaid
flowchart TB
    subgraph AnchorLanes[Anchor lanes]
      L1[Baseline lane<br/>num_batch=512]
      L2[Side lane<br/>num_batch=1024]
    end

    subgraph Tier1[Tier-1 shapes]
      S1[512x512x2880 / 512x1024x2880]
      S2[2880x512x4096 / 2880x1024x4096]
      S3[4096x512x2880 / 4096x1024x2880]
    end

    subgraph Candidates[Shared lane-level candidate set]
      K1[Cijk_BBS_BH_*]
      K2[Cijk_HB_GB_*]
      K3[Cijk_HSS_BH_GB_*]
      K4[Cijk_SB_*_ISA900]
    end

    subgraph Assets[HSACO mapping]
      H1[Type_BB_HPA_fallback_gfx900.hsaco]
      H2[Type_HH_fallback_gfx900.hsaco]
      H3[Type_HS_HPA_fallback_gfx900.hsaco]
      H4[ISA900 candidate<br/>unmatched]
    end

    L1 --> S1
    L1 --> S2
    L1 --> S3
    L2 --> S1
    L2 --> S2
    L2 --> S3

    S1 --> K1
    S2 --> K2
    S3 --> K3
    S1 -. lane-level only .-> K4

    K1 --> H1
    K2 --> H2
    K3 --> H3
    K4 --> H4
```

## 4. Evidence Class Guardrail

- `prefill/full shape proxy`:
  - good for shape-count deltas
  - not sufficient alone for token-level dispatch attribution
- `stream phase-window`:
  - complementary decode-side proxy
  - use together with gate fields (`direct/fallback/dispatch`)
- `candidate -> hsaco`:
  - links likely kernel family to concrete files
  - still requires careful per-shape attribution evidence

## 5. Current Status (Fact vs Inference)

- [main-node confirmed] baseline and side lanes both keep `direct/fallback/dispatch=1`.
- [main-node confirmed] stream phase-window is `decode_signature_detected` in both lanes.
- [main-node confirmed] Cijk candidate mapping remains `4 -> 3 matched + 1 unmatched`.
- [inference] per-shape attribution is still lane-level correlated, not yet single-shape dispatch proof.

## 6. References

- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022355.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022435.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022531.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022637.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022802.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022953.txt`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331.tsv`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347_20260325_023354.txt`
