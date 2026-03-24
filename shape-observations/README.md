# rocBLAS gfx900 Shape Observations

Last updated: 2026-03-25

This directory stores per-shape notes extracted from the current
`gpt-oss` direct-dispatch anchor lane on MI25/gfx900.

Source class labels:

- `[main-node confirmed]` runtime evidence captured from the current node.
- `[inference]` interpretation based on the captured evidence.

Current Tier-1 notes:

- `shape_512x512x2880.md`
- `shape_2880x512x4096.md`
- `shape_4096x512x2880.md`
- `tier1_shape_correlation_map.md` (Mermaid map)

Queue-B notes:

- `shape_512x93x2880.md`
- `shape_32x512x2880.md`

Queue-C notes:

- `shape_4608x512x64.md`
- `shape_64x512x4608.md`
- `shape_8192x512x64.md`
- `shape_64x512x8192.md`

Common probe references (2026-03-25):

- baseline prefill/decode split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_011553.txt`
- side prefill/decode split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_011411.txt`

## Observation refresh (2026-03-25 02:23 JST)

This cycle is observation-only (no kernel/source modifications).
Goal: increase evidence granularity for top-shape lanes.

Anchor re-observation:

- baseline lane (`num_batch=512`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022355.txt`
- side lane (`num_batch=1024`):
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_gptoss_anchor_shape_sweep_gpt-oss_latest_20260325_022435.txt`

Prefill/full split (shape-proxy):

- baseline split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022531.txt`
- side split:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_prefill_decode_split_gpt-oss_latest_20260325_022637.txt`

Stream phase-window split:

- baseline sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022802.txt`
- side sweep:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/g4_stream_phase_window_sweep_gpt-oss_latest_20260325_022953.txt`

Candidate to HSACO to disasm (same cycle):

- kernel candidates:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311.txt`
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022651__rocprofv3_summary_gpt-oss_latest_20260325_022727_20260325_023315.txt`
- map:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331.txt`
- extract:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347.txt`
- disasm signals:
  - `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/disasm_signal_summary_hsaco_targets_hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331_20260325_023347_20260325_023354.txt`

Note on evidence class:

- `g4_prefill_decode_split_*` is shape-proxy (`full - prefill_proxy`) and should not be
  interpreted alone as token-level dispatch attribution.
- `g4_stream_phase_window_sweep_*` provides stream-window proxy visibility and is used as
  a complementary decode-side signal.
