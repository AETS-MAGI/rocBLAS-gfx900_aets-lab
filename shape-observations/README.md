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
