# Shape Kernel Priority Memo: 32x512x2880 (Queue-B)

Last updated: 2026-03-25
Class: observation-only (no code/asset modification)

## 1. Scope

Target shape pair:

- baseline: `32x512x2880`
- side counterpart: `32x1024x2880`

## 2. Stable observation snapshot

- [main-node confirmed] baseline lane:
  - `shape_32x512x2880=46` (`prefill -> full: 46 -> 46`, `delta=0`)
  - `direct/fallback/dispatch=1`
- [main-node confirmed] side lane:
  - `shape_32x1024x2880=69` (`prefill -> full: 69 -> 69`, `delta=0`)
  - `direct/fallback/dispatch=1`
- [main-node confirmed] stream window (both lanes):
  - `decode_signature_detected` (3/3 rows)

## 3. Candidate priority (lane-level shared set)

| Priority | Kernel ID | Candidate family | prefill/full count | HSACO map |
|:--|:--|:--|:--|:--|
| P1 | K1 | `Cijk_*_BBS_BH_*` | `96 / 96` | matched (`Type_BB_HPA`) |
| P2 | K2 | `Cijk_*_HB_GB_*` | `24 / 24` | matched (`Type_HH`) |
| P2 | K3 | `Cijk_*_HSS_BH_GB_*` | `24 / 24` | matched (`Type_HS_HPA`) |
| P3 | K4 | `Cijk_*_SB_*_ISA900` | `23 / 23` | unmatched (current `*gfx900*.hsaco` scan) |

## 4. Visual map

```mermaid
flowchart LR
    S[Shape pair<br/>32x512x2880 / 32x1024x2880] --> K1[K1: BBS_BH]
    S --> K2[K2: HB_GB]
    S --> K3[K3: HSS_BH_GB]
    S -. lane-level candidate .-> K4[K4: SB_ISA900]

    K1 --> H1[Type_BB_HPA_fallback_gfx900.hsaco]
    K2 --> H2[Type_HH_fallback_gfx900.hsaco]
    K3 --> H3[Type_HS_HPA_fallback_gfx900.hsaco]
    K4 --> HU[unmatched in current gfx900 hsaco set]
```

## 5. Guardrail

- [inference] This memo is queue-B shape prioritization, not single-shape dispatch proof.
- Keep `prefill/full proxy` and `stream-window proxy` separated.

## 6. References

- `/home/limonene/ROCm-project/ROCm-repos_AETS/rocBLAS/shape-observations/shape_32x512x2880.md`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311.tsv`
- `/home/limonene/ROCm-project/vega_path_check_logs_raw/summaries/hsaco_candidate_map_kernel_candidates_rocprofv3_summary_gpt-oss_latest_20260325_022545__rocprofv3_summary_gpt-oss_latest_20260325_022614_20260325_023311_20260325_023331.tsv`
