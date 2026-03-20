# rocBLAS

[English](README.md) | [日本語](README.ja.md)

> [!IMPORTANT]
> This repository is an experimental lab fork: `AETS-MAGI/rocBLAS-gfx900_aets-lab`.
>
> - Fork source: `ROCm/rocBLAS`
> - Purpose: gfx900/MI25 bring-up and ROCm runtime investigation
> - Branch policy: `main` as baseline, experiments on dedicated branches (for example `gfx900-bringup`)
> - License note: upstream `LICENSE` and copyright notices are preserved

> [!CAUTION]
> The rocBLAS repository is retired, please use the [ROCm/rocm-libraries](https://github.com/ROCm/rocm-libraries) repository

rocBLAS is the [ROCm](https://rocm.docs.amd.com/en/latest) Basic Linear Algebra Subprograms (BLAS)
library. rocBLAS is implemented in the
[HIP programming language](https://github.com/ROCm/HIP) and optimized for AMD
GPUs.

## Requirements

You must have ROCm installed on your system before you can install rocBLAS. For information on
ROCm installation and required platform dependencies, refer to the
[ROCm](https://rocm.docs.amd.com/en/latest).

## Documentation

> [!NOTE]
> The published rocBLAS documentation is available at [rocBLAS](https://rocm.docs.amd.com/projects/rocBLAS/en/latest/index.html) in an organized, easy-to-read format, with search and a table of contents. The documentation source files reside in the rocBLAS/docs folder of this repository. As with all ROCm projects, the documentation is open source. For more information, see [Contribute to ROCm documentation](https://rocm.docs.amd.com/en/latest/contribute/contributing.html).

## Related repositories

- Setup and validation workspace: https://github.com/AETS-MAGI/ROCm-MI25-build
- Paired Tensile fork: https://github.com/AETS-MAGI/Tensile-gfx900_aets-lab
