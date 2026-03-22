# Ollama gfx900 Starter Kit

<p align="center">
  <img src="assets/img/logo.webp" alt="Ollama gfx900 Starter Kit" width="200"/>
</p>

Pre-built backend libraries for running [Ollama](https://ollama.com) on **AMD Vega / gfx900 / Radeon Instinct MI25** hardware.

English | [日本語](README.ja.md)

Upstream Ollama does not support gfx900. This kit provides the compiled HIP/ROCm backend that enables inference on MI25.

> **Scope:** Linux + ROCm only. Windows HIP excludes gfx900 upstream and is not supported here.

## Repository Role Split

- `ollama-gfx900-starter-kit` is the distribution repository for prebuilt artifacts.
- `ollama-src` is the source-of-truth for patches, implementation details, and validation notes.
- Cleanup policy is non-destructive: keep investigation history in source, and publish user-facing binaries via starter-kit releases.

---

## What's Included

```
build-gfx900/
└── lib/
    └── ollama/
        ├── libggml-base.so(.0/.0.0.0)     # GGML base
        ├── libggml-cpu-*.so                # CPU variants (haswell / sandybridge / sse42 / ...)
        └── libggml-hip.so                  # HIP/ROCm backend compiled for gfx900
```

These libraries work together with a gfx900-patched `ollama` binary (see [Getting the Binary](#getting-the-binary)).

---

## Requirements

| Component | Requirement |
|-----------|-------------|
| GPU | AMD Radeon Instinct MI25 (gfx900) |
| OS | Linux (Ubuntu 22.04 / 24.04 recommended) |
| ROCm | 6.1 or later (`amdgpu-install`) |
| glibc | 2.35+ (Ubuntu 22.04 baseline) |

Verify your GPU is recognized:

```bash
rocminfo | grep -E "Name|gfx"
# Expected: a line containing "gfx900"
```

---

## Getting the Binary

The `ollama` binary is not bundled here. Obtain it in one of two ways:

**Option A — Download pre-built binary (recommended)**

Download `ollama` from the [Releases](../../releases) page of this repository and place it in the same directory as `build-gfx900/`.

**Option B — Build from source**

```bash
git clone https://github.com/<your-org>/ollama-src
cd ollama-src
git checkout vega-int8-probe

# Build HIP backend for gfx900
cmake -B build-gfx900 \
    -DAMDGPU_TARGETS=gfx900 \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build-gfx900 --parallel $(nproc)

# Build the ollama binary
go build -o ollama .
```

---

## Setup

### 1. Place the starter kit

```bash
# Example layout (any directory works)
~/ollama-gfx900/
├── ollama                  # The patched binary
└── build-gfx900/
    └── lib/
        └── ollama/
            ├── libggml-base.so*
            ├── libggml-cpu-*.so
            └── libggml-hip.so
```

The runtime searches for `build-gfx900/lib/ollama/` relative to both the binary's directory and the current working directory — no extra configuration needed.

### 2. (Optional) Restrict to MI25 only

If your system has multiple GPUs, pin Ollama to the MI25:

```bash
# Get the MI25 UUID
rocminfo | grep -A5 "gfx900" | grep "Uuid"

# Set it before starting
export ROCR_VISIBLE_DEVICES=<UUID>
```

### 3. Start Ollama

```bash
cd ~/ollama-gfx900
./ollama serve
```

Confirm gfx900 detection in the logs:

```
level=INFO msg="forcing num_parallel=1 for ROCm gfx900" compute=gfx900
```

---

## Runtime Behavior on gfx900

These behaviors are applied automatically when a gfx900 device is detected — no manual environment variables required.

| Setting | Value | Reason |
|---------|-------|--------|
| Flash Attention | Disabled | Not supported on gfx900 |
| `num_parallel` | 1 (forced) | Stability on gfx900 |
| `num_ctx` default | 4096 (typical on MI25 16 GB) | VRAM-based default; capped at 8192 max |

The log will show `FlashAttention:Auto` — this is expected behavior; Flash Attention is not used regardless of `OLLAMA_FLASH_ATTENTION`.

---

## Pull and Run a Model

```bash
./ollama pull tinyllama
./ollama run tinyllama
```

For context-length-sensitive workloads, set `num_ctx` explicitly (max 8192 on MI25):

```bash
./ollama run --num-ctx 4096 tinyllama
```

---

## rocBLAS / Tensile Kernels

The system ROCm package may not include gfx900-specific Tensile kernels, which can affect GEMM performance. If you observe degraded performance or rocBLAS initialization errors, build local kernels using the scripts in [ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build):

```bash
./build-rocblas-gfx900.sh
export ROCBLAS_TENSILE_LIBPATH=/path/to/local/rocblas/library
```

---

## Troubleshooting

**Ollama falls back to CPU**

Check that `rocminfo` lists `gfx900` and that `ROCR_VISIBLE_DEVICES` is not set to an invalid value.

**`libggml-hip.so: cannot open shared object`**

Ensure `build-gfx900/lib/ollama/` is in the same directory tree as the binary. The runtime looks for it relative to the binary location and the current working directory.

**rocBLAS initialization failure**

Build gfx900-specific Tensile kernels via [ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build) and set `ROCBLAS_TENSILE_LIBPATH`.

**Flash Attention log shows `FlashAttention:Auto`**

This is expected. Flash Attention is disabled in code for gfx900 regardless of the `OLLAMA_FLASH_ATTENTION` environment variable.

---

## Source Repository

The patched Ollama source is maintained at:
[ollama-src / vega-int8-probe branch](../../)

Key patches applied:

| Change | File |
|--------|------|
| `IsROCmGFX900()` detection | `ml/device.go` |
| Flash Attention disabled for gfx900 | `ml/device.go` |
| `num_parallel=1` forced for gfx900 | `server/sched.go` |
| `num_ctx` capped at 8192 for gfx900 | `server/routes.go` |
| `build-gfx900/lib/ollama` prioritized in path search | `ml/path.go` |
| gfx900 added to CMake ROCm presets | `CMakePresets.json` |
| Dockerfile rocBLAS cleanup narrowed to gfx906 only | `Dockerfile` |

---

## Disclaimer

This is an experimental fork for investigation and validation purposes on Vega / gfx900 / MI25 hardware.
It is not an official compatibility statement, product commitment, or support guarantee.

End-to-end success should only be claimed when directly verified for the target model and workload on the target hardware.
