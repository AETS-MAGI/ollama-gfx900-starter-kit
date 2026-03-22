# Release Notes — ollama-gfx900-starter-kit

**Base:** Ollama v0.18.2
**Branch:** `vega-int8-probe`
**Build tag:** `v0.18.2-20-gd44f21d89`
**Target:** AMD Radeon Instinct MI25 / gfx900 (Vega10)
**Platform:** Linux + ROCm 6.1+

---

## What This Release Adds

Upstream Ollama blocks gfx900 at runtime (commit `55ca82726`). This kit removes that block and provides a HIP backend compiled specifically for gfx900, enabling LLM inference on MI25 hardware.

### Patches Applied (20 commits over v0.18.2)

| Change | File |
|--------|------|
| `IsROCmGFX900()` detection function | `ml/device.go` |
| Flash Attention disabled for gfx900 | `ml/device.go` |
| `num_parallel=1` forced for gfx900 | `server/sched.go` |
| `num_ctx` capped at 8192 for gfx900 | `server/routes.go` |
| `build-gfx900/lib/ollama` prioritized in library path search | `ml/path.go` |
| gfx900 added to CMake ROCm presets | `CMakePresets.json` |
| Dockerfile rocBLAS cleanup narrowed to gfx906 only | `Dockerfile` |

---

## Included Artifacts

`build-gfx900/lib/ollama/` contains:

| Library | Description |
|---------|-------------|
| `libggml-base.so` / `.so.0` / `.so.0.0.0` | GGML base |
| `libggml-hip.so` | HIP/ROCm backend compiled for gfx900 |
| `libggml-cpu-alderlake.so` | CPU backend (Alder Lake) |
| `libggml-cpu-haswell.so` | CPU backend (Haswell) |
| `libggml-cpu-icelake.so` | CPU backend (Ice Lake) |
| `libggml-cpu-sandybridge.so` | CPU backend (Sandy Bridge) |
| `libggml-cpu-skylakex.so` | CPU backend (Skylake-X) |
| `libggml-cpu-sse42.so` | CPU backend (SSE4.2) |
| `libggml-cpu-x64.so` | CPU backend (generic x86-64) |

The `ollama` binary is distributed separately on the [Releases](../../releases) page.

---

## Known Limitations

**Flash Attention**
Flash Attention is not supported on gfx900. It is disabled in code regardless of the `OLLAMA_FLASH_ATTENTION` environment variable. The log will show `FlashAttention:Auto` — this is expected.

**Concurrency**
`num_parallel` is forced to 1. Concurrent request handling is not supported on this hardware target.

**Context length**
`num_ctx` is capped at 8192. The practical default on a 16 GB MI25 is 4096 depending on model size.

**rocBLAS / Tensile kernels**
The system ROCm package may not include gfx900-specific Tensile kernels. If GEMM performance is degraded or rocBLAS initialization fails, build local kernels from [ROCm-MI25-build](https://github.com/AETS-MAGI/ROCm-MI25-build).

**Windows**
Not supported. The upstream Windows HIP build excludes gfx900, and no workaround is provided here.

**MLIR iGEMM / XDLops**
These compute paths are not available on gfx900. The active paths are ASM implicit GEMM, DLOPS, and non-dot4 fallback routes.

---

## Verification

After setup, confirm GPU detection:

```bash
./ollama serve
# Expected log line:
# level=INFO msg="forcing num_parallel=1 for ROCm gfx900" compute=gfx900
```

Quick functional check:

```bash
./ollama pull tinyllama
./ollama run tinyllama "Hello"
```

---

## Disclaimer

This is an experimental release for investigation and validation on Vega / gfx900 / MI25 hardware. It is not an official compatibility statement, product commitment, or support guarantee from AMD or the Ollama project.

End-to-end success should only be claimed when directly verified for the target model and workload on the target hardware.
