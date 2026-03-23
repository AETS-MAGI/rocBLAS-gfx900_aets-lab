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
