# Compile llama.cpp from source (Ubuntu 26.04 WSL2, NVIDIA CUDA)

## Goal

Build `llama-server`, `llama-cli`, and `llama-completion` from the official `ggml-org/llama.cpp` repository at the pinned tag `gguf-v0.19.0`, with CUDA acceleration targeted to the NVIDIA GPUs that `detect.md` already reported. Produces runnable binaries at `src/llama.cpp/build/bin/`.

## Pre-conditions

The agent must have already executed `references/detect.md` and accumulated the following facts in its context. If any of them is missing or contradicts what is below, abort with a clear message and re-invoke the skill from the top.

| Field | Expected value | Source |
| --- | --- | --- |
| Selected recipe folder | `references/ubuntu-26.04-wsl2/` | `detect.md` step "Recipe selection" |
| `distro_id` | `ubuntu` | `detect.md` step A |
| `distro_version` | `26.04` | `detect.md` step A |
| `gpu_vendor` | `nvidia` (or any non-empty value) | `detect.md` step E |
| `gpus[]` | one entry per NVIDIA device, each with `index`, `name`, `compute_cap`, `memory.total` | `detect.md` step E |
| `nvidia_runtime_registered` | `true` | `detect.md` step F |
| `verdict` | `ready` or `ready with warnings` | `detect.md` |
| `nvcc_present` | `true` | **this recipe** (re-checked) |
| `cmake_present` | `true` | **this recipe** (re-checked) |
| `nproc` | integer | **this recipe** (re-checked) |

This recipe does **not** re-run `detect.md`. It assumes the previous run already produced the data. If the agent is invoked fresh (no `detect.md` context), abort with: "This recipe expects a prior `references/detect.md` run. Re-invoke the skill from the top."

## Why build from source

The user already chose the "build from source" path during the entry-point gate. The pre-built image `ghcr.io/ggml-org/llama.cpp:server-cuda` is the alternative and is documented in `references/ubuntu-26.04-wsl2/docker/llama-cpp.md` (forthcoming). Do not switch paths silently.

## A. Re-validate the build prerequisites

These checks are cheap and protect against the host being modified between the `detect.md` run and this step. The agent must run them and abort on any failure.

```bash
command -v cmake >/dev/null 2>&1 && echo "cmake_present=true" || echo "cmake_present=false"
command -v nvcc >/dev/null 2>&1 && echo "nvcc_present=true" || echo "nvcc_present=false"
nproc
nvcc --version | tail -1
```

Interpretation rules:

- `cmake_present=false` → abort with: "`cmake` is not installed. Install with the distro's package manager (e.g. `sudo apt install -y cmake`) and re-invoke the skill from the top."
- `nvcc_present=false` → abort with: "The NVIDIA CUDA Toolkit (`nvcc`) is not installed. This is required to compile `llama.cpp` with CUDA support. Install CUDA 12.8 or newer for the host (the official `.run` installer from NVIDIA is the most reliable path on WSL2; distro packages are often behind). Alternatively, switch to the pre-built image path: re-invoke the skill and choose the Docker image option at the entry-point gate."
- Capture `nvcc --version` and record the CUDA version in the build report (it determines which `sm_XX` targets are usable).

## B. Derive the CUDA architecture list

```bash
nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits | awk '!seen[$0]++' | paste -sd ';'
```

This produces a deduplicated, semicolon-separated list of the compute capabilities present in the host (e.g. `12.0;8.9`). The output is the value of `CMAKE_CUDA_ARCHITECTURES`. The agent must capture it and pass it verbatim to `cmake` in step D. Do not invent values; do not hard-code.

If the output is empty (no `nvidia-smi` rows), abort with: "No NVIDIA GPUs reported by `nvidia-smi`. The CUDA build needs at least one. Re-invoke the skill after fixing the host."

## C. Clone or update the pinned source tree

The source tree lives at `src/llama.cpp/` at the repo root. If the folder is absent, clone it. If it is present, fetch and check the pinned tag.

```bash
test -d src/llama.cpp || git clone --depth 1 https://github.com/ggml-org/llama.cpp.git src/llama.cpp
cd src/llama.cpp
git fetch --tags --depth 1 origin
git checkout gguf-v0.19.0
```

After the checkout, verify that the working tree matches the pinned commit and abort if it does not:

```bash
EXPECTED_SHA="a290ce626663dae1d54f70bce3ca6d8f67aab62f"
ACTUAL_SHA="$(git rev-parse HEAD)"
[ "$ACTUAL_SHA" = "$EXPECTED_SHA" ] || { echo "Expected $EXPECTED_SHA, got $ACTUAL_SHA"; exit 1; }
```

The exact commit SHA is what makes the build reproducible. If the pin changes in a future release, update the recipe in the same commit that bumps the version; do not let the two drift.

## D. Configure with CMake

Working directory: `src/llama.cpp/`. The variable `<CUDA_ARCHES>` is the value produced by step B (e.g. `12.0;8.9`).

```bash
cmake -B build -S . \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES="<CUDA_ARCHES>" \
  -DCMAKE_BUILD_TYPE=Release
```

The agent must use `-DCMAKE_BUILD_TYPE=Release`. Debug builds produce binaries that are too slow for inference and are not part of this skill.

The output of `cmake` configure is verbose. The agent must scan it for any of these failure indicators and abort on any match:

- `Could NOT find CUDA` → abort: "CMake did not find CUDA. Verify that the toolkit is installed and that `nvcc` is on `PATH`."
- `CUDA error` → abort: "CMake configure failed with a CUDA error. Read the lines above for the specific cause."
- `No CUDA arch specified` → abort: "CMake did not detect any CUDA architecture. Pass `-DCMAKE_CUDA_ARCHITECTURES` explicitly with the value from step B."

If the configure finishes without those errors, capture the line that starts with `-- Configuring done` as the success marker.

## E. Build

Working directory: `src/llama.cpp/`. The variable `<N>` is the value of `nproc` captured in step A.

```bash
cmake --build build --config Release -j <N>
```

The `-j <N>` is mandatory. Without it, the build will appear to hang on a many-core machine (the AMD Ryzen 7 7800X3D has 16 threads, and a default serial build of `llama.cpp` with CUDA kernels can take 30+ minutes).

The build output is large. The agent must wait for it to finish. If the build exits non-zero, abort with: "Build failed. Read the last 50 lines of output for the failing target and report the error to the user. Do not retry automatically."

## F. Verify the binaries

```bash
test -x build/bin/llama-server   || { echo "missing: build/bin/llama-server"; exit 1; }
test -x build/bin/llama-cli      || { echo "missing: build/bin/llama-cli"; exit 1; }
test -x build/bin/llama-completion || { echo "missing: build/bin/llama-completion"; exit 1; }

build/bin/llama-server --version
build/bin/llama-cli --version
```

The `--version` output must show the same `gguf-v0.19.0` version the source tree is pinned to. If the version printed does not match the pin, the build is from the wrong source: abort and re-do steps C and D.

## G. Report to the user

After successful verification, present:

```
Compile report
--------------
Source tree:    src/llama.cpp/
Pin:            gguf-v0.19.0 (commit a290ce626663dae1d54f70bce3ca6d8f67aab62f)
CUDA toolkit:   <output of `nvcc --version`>
CUDA arch list: <value from step B>
CMake:          <output of `cmake --version` first line>
Binaries:
  build/bin/llama-server     <size>   <sha256 prefix>
  build/bin/llama-cli        <size>   <sha256 prefix>
  build/bin/llama-completion <size>   <sha256 prefix>

Verdict: ready
```

Generate the size and sha256 lines with:

```bash
ls -l build/bin/llama-server build/bin/llama-cli build/bin/llama-completion | awk '{print $5, $NF}'
sha256sum build/bin/llama-server build/bin/llama-cli build/bin/llama-completion | awk '{print $1, $NF}'
```

The "Verdict" here is the compile step's verdict, independent of `detect.md`'s verdict. If any verification in steps A–F failed and the recipe aborted, do not print this report.

## What this file does NOT do

- It does not download models. That is the responsibility of `references/ubuntu-26.04-wsl2/models/download.md` (forthcoming).
- It does not build a Docker image. That is the responsibility of `references/ubuntu-26.04-wsl2/docker/llama-cpp.md` (forthcoming).
- It does not start the server. That is the responsibility of `references/ubuntu-26.04-wsl2/compose.md` (forthcoming).
- It does not modify any file outside `src/llama.cpp/`. In particular, it does not install system packages; the user must do that and re-invoke the skill from the top.

## Inputs this recipe expects from the prior agent context

Re-stated for clarity so the subagent (or the human running the skill) can sanity-check before executing:

1. The selected recipe folder from `detect.md` (must be `references/ubuntu-26.04-wsl2/`).
2. The list of NVIDIA GPUs with their `compute_cap` values.
3. The `nproc` value.
4. The presence of `nvcc` and `cmake` on `PATH`.

If any of these is missing, do not guess. Abort with a clear message and tell the user to re-invoke the skill from the top.
