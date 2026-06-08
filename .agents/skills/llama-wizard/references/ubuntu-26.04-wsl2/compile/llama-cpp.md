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
```

Interpretation rules:

- `cmake_present=false` → abort with: "`cmake` is not installed. Install with the distro's package manager (e.g. `sudo apt install -y cmake`) and re-invoke the skill from the top."
- `nvcc_present=false` → abort with: "The NVIDIA CUDA Toolkit (`nvcc`) is not installed. This recipe needs it to build `llama.cpp` with CUDA support. To install, follow the official instructions at https://developer.nvidia.com/cuda-downloads (the page lets you pick your distro and version). The user must run the install manually; re-invoke the skill afterwards. If you do not want to install the toolkit, switch to the pre-built image path: use the official image `ghcr.io/ggml-org/llama.cpp:server-cuda` directly (no local compile needed)."
- **Once an abort condition is met, stop. Do not run the remaining commands in this block, and do not try `nvcc --version` against a missing `nvcc` — the abort already covers the case.**

## B. Derive the CUDA architecture list

```bash
nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits \
  | awk -F. '{printf "%d%d\n", $1, $2}' \
  | awk '!seen[$0]++' \
  | paste -sd ';'
```

This produces a deduplicated, semicolon-separated list of compute capabilities in the **integer form** CMake requires (e.g. `120;89`, not `12.0;8.9`). The output is the value of `CMAKE_CUDA_ARCHITECTURES`. The agent must capture it and pass it verbatim to `cmake` in step D. Do not invent values; do not hard-code; do not keep the `X.Y` form from `nvidia-smi` — CMake will reject it.

If the output is empty (no `nvidia-smi` rows), abort with: "No NVIDIA GPUs reported by `nvidia-smi`. The CUDA build needs at least one. Re-invoke the skill after fixing the host."

## C. Clone or update the pinned source tree

The source tree lives at `src/llama.cpp/` at the repo root. If the folder is absent, clone it. If it is present, fetch and check the pinned tag.

```bash
test -d src/llama.cpp || git clone --depth 1 https://github.com/ggml-org/llama.cpp.git src/llama.cpp
cd src/llama.cpp
git fetch --tags --depth 1 origin
git checkout gguf-v0.19.0
```

After the checkout, verify that the working tree is on the pinned tag and abort if it is not:

```bash
TAG_REF="$(git rev-parse --verify 'refs/tags/gguf-v0.19.0^{commit}' 2>/dev/null)"
ACTUAL_SHA="$(git rev-parse HEAD)"
[ "$ACTUAL_SHA" = "$TAG_REF" ] || { echo "Expected $TAG_REF (tag gguf-v0.19.0), got $ACTUAL_SHA"; exit 1; }
```

The pin is the **tag**, not a hard-coded SHA. The check uses `^{commit}` to peel the tag object to the commit it points to — this is required because `gguf-v0.19.0` (and every llama.cpp release tag) is an **annotated** tag, and `git rev-parse refs/tags/...` returns the tag-object SHA, not the commit SHA, in that case. Without the peel, the check fails on a correct checkout. The check verifies that the working tree's HEAD is the commit the tag points to. If the project releases `gguf-v0.19.1` later, only the `git checkout` line above needs to change; this check keeps working without any edit.

## D. Configure with CMake

This step's `cmake` invocation must run with `src/llama.cpp/` as the current directory. The agent's `bash` tool typically does not preserve `cwd` between separate calls, so the `cd` must be in the same command line. The variable `<CUDA_ARCHES>` is the value produced by step B in **integer form** (e.g. `120;89`, not `12.0;8.9`).

```bash
cd src/llama.cpp && cmake -B build -S . \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES="<CUDA_ARCHES>" \
  -DCMAKE_BUILD_TYPE=Release
```

The agent must use `-DCMAKE_BUILD_TYPE=Release`. Debug builds produce binaries that are too slow for inference and are not part of this skill.

The output of `cmake` configure is verbose. The agent must scan it for any of these failure indicators and abort on any match:

- `source directory` followed by `does not appear to contain CMakeLists.txt` → abort: "The `cd src/llama.cpp` did not take effect (or the directory was renamed). The bash tool may have reset the working directory. Re-run with `cd` in the same command line as `cmake`."
- `Could NOT find CUDA` → abort: "CMake did not find CUDA. Verify that the toolkit is installed and that `nvcc` is on `PATH`."
- `CMAKE_CUDA_ARCHITECTURES` followed by `is not one of the following` → abort: "CMake rejected `CMAKE_CUDA_ARCHITECTURES`. The value must be a semicolon-separated list of integers (e.g. `120;89`), not the `X.Y` form from `nvidia-smi`. Re-check step B."
- `CUDA error` → abort: "CMake configure failed with a CUDA error. Read the lines above for the specific cause."
- `No CUDA arch specified` → abort: "CMake did not detect any CUDA architecture. Pass `-DCMAKE_CUDA_ARCHITECTURES` explicitly with the value from step B."

If the configure finishes without those errors, capture the line that starts with `-- Configuring done` as the success marker.

## E. Build

The variable `<N>` is the value of `nproc` captured in step A. As in step D, the `cd` must be in the same command line because the agent's `bash` tool does not preserve `cwd` between calls.

```bash
cd src/llama.cpp && cmake --build build --config Release -j <N>
```

The `-j <N>` is mandatory. Without it, the build will appear to hang on a many-core machine (the AMD Ryzen 7 7800X3D has 16 threads, and a default serial build of `llama.cpp` with CUDA kernels can take several hours).

**Expected duration on this hardware**: 60-120 minutes for the first build with `-DGGML_CUDA=ON` and two compute capabilities (e.g. `120;89`). The bottleneck is `nvcc` compiling the `ggml-cuda` source tree, which has hundreds of templated `.cu` files (one per fattn and mmq variant per arch). The `-j 16` parallelism helps with `cc1plus` jobs but `nvcc` jobs serialize around the GPU. Set the bash tool's foreground timeout to at least 3 hours (10_800_000 ms) for this step.

**If the bash tool's timeout is too short for a single foreground call, run the build in the background and monitor it.** This is the documented workaround for long builds, not improvisation:

```bash
cd src/llama.cpp && nohup cmake --build build --config Release -j <N> > /tmp/llama-build.log 2>&1 &
echo "BUILD_PID=$!"
```

Then poll the log and the process list every 60-120 seconds:

```bash
sleep 90 && tail -3 /tmp/llama-build.log && echo "---" && \
  ps -ef | grep -E "cmake|cc1plus|nvcc" | grep -v grep | wc -l && echo "active build procs" && \
  ls src/llama.cpp/build/bin/ 2>/dev/null | wc -l && echo "binaries so far"
```

The build is complete when `[100%] Built target llama-server` (or equivalent) appears in the log, AND the process list shows no active `nvcc`/`cc1plus`/`cmake` processes, AND `ls src/llama.cpp/build/bin/` shows the three expected binaries (`llama-server`, `llama-cli`, `llama-completion`).

The build output is large (10+ MB of compile logs). The agent must wait for it to finish. If the build exits non-zero, abort with: "Build failed. Read the last 50 lines of output for the failing target and report the error to the user. Do not retry automatically."

## F. Verify the binaries

```bash
test -x build/bin/llama-server   || { echo "missing: build/bin/llama-server"; exit 1; }
test -x build/bin/llama-cli      || { echo "missing: build/bin/llama-cli"; exit 1; }
test -x build/bin/llama-completion || { echo "missing: build/bin/llama-completion"; exit 1; }

build/bin/llama-server --version
build/bin/llama-cli --version
```

The `--version` output does NOT print a `gguf-vX.Y.Z` string; it prints `version: 1 (<short commit SHA>)`. Verify the build is from the pinned source by comparing the printed short SHA against the tag's peeled commit:

```bash
TAG_SHA="$(git -C src/llama.cpp rev-parse --short 'refs/tags/gguf-v0.19.0^{commit}')"
SERVER_SHA="$(build/bin/llama-server --version 2>&1 | grep -Eo '\([0-9a-f]+\)' | tr -d '()' | head -1)"
CLI_SHA="$(build/bin/llama-cli --version 2>&1 | grep -Eo '\([0-9a-f]+\)' | tr -d '()' | head -1)"
[ "$SERVER_SHA" = "$TAG_SHA" ] || { echo "llama-server commit $SERVER_SHA does not match tag gguf-v0.19.0 ($TAG_SHA)"; exit 1; }
[ "$CLI_SHA" = "$TAG_SHA" ] || { echo "llama-cli commit $CLI_SHA does not match tag gguf-v0.19.0 ($TAG_SHA)"; exit 1; }
```

If either binary's printed short SHA does not match the pinned tag's peeled commit, the build is from the wrong source. Abort with: "Binary commit does not match the pinned tag `gguf-v0.19.0` ($TAG_SHA). Re-do steps C and D from a clean clone."

## G. Report to the user

After successful verification, present:

```
Compile report
--------------
Source tree:    src/llama.cpp/
Pin:            gguf-v0.19.0 (commit resolved at build time via `git rev-parse refs/tags/gguf-v0.19.0^{commit}`)
CUDA toolkit:   Cuda compilation tools, release <major>.<minor>, V<release>   (first line of `nvcc --version`)
CUDA arch list: <value from step B, in integer form>
CMake:          <output of `cmake --version | head -1`>   (e.g. `cmake version 4.2.3`)
Binaries:
  build/bin/llama-server     <size>   <sha256 prefix>
  build/bin/llama-cli        <size>   <sha256 prefix>
  build/bin/llama-completion <size>   <sha256 prefix>

Compile verdict: ready
```

Generate the size and sha256 lines with:

```bash
ls -l build/bin/llama-server build/bin/llama-cli build/bin/llama-completion | awk '{print $5, $NF}'
sha256sum build/bin/llama-server build/bin/llama-cli build/bin/llama-completion | awk '{print $1, $NF}'
```

The `nvcc --version` output has multiple lines (header, copyright, build date, version, build path). Capture only the line that contains the version string (it starts with `Cuda compilation tools, release ...`). The full multi-line output is not part of the report — it goes in the build log if the user wants to inspect it.

The "Compile verdict" here is the compile step's verdict, independent of `detect.md`'s verdict. It means "the binaries exist and were built from the pinned source tree"; it does not mean the server is running or that any model is loaded. If any verification in steps A–F failed and the recipe aborted, do not print this report.

## What this file does NOT do

- It does not download models. That is out of scope for this recipe.
- It does not build a Docker image. That is also out of scope; for the pre-built image path, the user can run `ghcr.io/ggml-org/llama.cpp:server-cuda` directly.
- It does not start the server. That is out of scope; the binary exists and the user can run it, but a full `docker compose` or systemd setup is not part of this recipe.
- It does not modify any file outside `src/llama.cpp/`. In particular, it does not install system packages; the user must do that and re-invoke the skill from the top.

## Inputs this recipe expects from the prior agent context

Re-stated for clarity so the subagent (or the human running the skill) can sanity-check before executing:

1. The selected recipe folder from `detect.md` (must be `references/ubuntu-26.04-wsl2/`).
2. The list of NVIDIA GPUs with their `compute_cap` values.
3. The `nproc` value.
4. The presence of `nvcc` and `cmake` on `PATH`.

If any of these is missing, do not guess. Abort with a clear message and tell the user to re-invoke the skill from the top.
