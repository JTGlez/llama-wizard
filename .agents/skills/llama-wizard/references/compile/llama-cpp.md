# Compile llama.cpp from source

## Goal

Build `llama-server`, `llama-cli`, and `llama-completion` from the official `ggml-org/llama.cpp` repository at the pinned tag `gguf-v0.19.0`, with CUDA acceleration targeted to the NVIDIA GPUs that `detect.md` already reported, and produce a Docker image (`llama-wizard-llama-cpp:gguf-v0.19.0`) that the upcoming `references/compose.md` will orchestrate.

The flow assumes Linux (gated by `detect.md`); the specific distro is irrelevant because the commands in this file (git, cmake, nvcc, docker, ldconfig) are present on every Linux distro with the prerequisites installed.

## Pre-conditions

The agent must have already executed `references/detect.md` and accumulated the following facts in its context. If any of them is missing or contradicts what is below, abort with a clear message and re-invoke the skill from the top.

| Field | Expected value | Source |
| --- | --- | --- |
| Selected flow | `references/compile/llama-cpp.md` | `detect.md` step "Flow selection" |
| `gpu_vendor` | `nvidia` (or any non-empty value) | `detect.md` step E |
| `gpus[]` | one entry per NVIDIA device, each with `index`, `name`, `compute_cap`, `memory.total` | `detect.md` step E |
| `nvidia_runtime_registered` | `true` | `detect.md` step F |
| `verdict` | `ready` or `ready with warnings` | `detect.md` |
| `nvcc_present` | `true` | **this flow** (re-checked) |
| `cmake_present` | `true` | **this flow** (re-checked) |
| `nproc` | integer | **this flow** (re-checked) |

This flow does **not** re-run `detect.md`. If the agent is invoked fresh (no `detect.md` context), abort with: "This flow expects a prior `references/detect.md` run. Re-invoke the skill from the top."

## A. Re-validate the build prerequisites

Run these checks; the host may have been modified between `detect.md` and this step.

```bash
command -v cmake >/dev/null 2>&1 && echo "cmake_present=true" || echo "cmake_present=false"
command -v nvcc >/dev/null 2>&1 && echo "nvcc_present=true" || echo "nvcc_present=false"
nproc
```

Interpretation:

- `cmake_present=false` → abort: "`cmake` is not installed. Install it with the host's package manager (the same one `detect.md` already probed; see its heuristic list) and re-invoke the skill from the top."
- `nvcc_present=false` → abort: "The NVIDIA CUDA Toolkit (`nvcc`) is not installed. This flow needs it. To install, follow the official instructions at https://developer.nvidia.com/cuda-downloads. The user must run the install manually; re-invoke the skill afterwards. If you do not want to install the toolkit, switch to the pre-built image path."
- **Once an abort condition is met, stop. Do not run the remaining commands in this block, and do not try `nvcc --version` against a missing `nvcc`.**

## B. Derive the CUDA architecture list

The agent must capture this output and pass it verbatim to step D. Do not invent values; do not hard-code; do not keep the `X.Y` form from `nvidia-smi` (CMake requires the integer form).

```bash
nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits \
  | awk -F. '{printf "%d%d\n", $1, $2}' \
  | awk '!seen[$0]++' \
  | paste -sd ';'
```

If the output is empty, abort: "No NVIDIA GPUs reported by `nvidia-smi`. The CUDA build needs at least one. Re-invoke the skill after fixing the host."

## C. Clone or update the pinned source tree

Idempotent: clones if absent, fetches and checks out the tag if present.

```bash
test -d src/llama.cpp || git clone --depth 1 https://github.com/ggml-org/llama.cpp.git src/llama.cpp
cd src/llama.cpp
git fetch --tags --depth 1 origin
git checkout gguf-v0.19.0
```

Verify the working tree is on the pinned tag. Use `^{commit}` to peel annotated tags (every llama.cpp release tag is annotated; without the peel, `git rev-parse refs/tags/X` returns the tag-object SHA, not the commit SHA).

```bash
TAG_REF="$(git rev-parse --verify 'refs/tags/gguf-v0.19.0^{commit}' 2>/dev/null)"
ACTUAL_SHA="$(git rev-parse HEAD)"
[ "$ACTUAL_SHA" = "$TAG_REF" ] || { echo "Expected $TAG_REF (tag gguf-v0.19.0), got $ACTUAL_SHA"; exit 1; }
```

If the project releases `gguf-v0.19.1` later, only the `git checkout` line above needs to change.

## D. Configure with CMake

`<CUDA_ARCHES>` is the value produced by step B (integer form, e.g. `120;89`).

```bash
cd src/llama.cpp && cmake -B build -S . \
  -DGGML_CUDA=ON \
  -DCMAKE_CUDA_ARCHITECTURES="<CUDA_ARCHES>" \
  -DCMAKE_BUILD_TYPE=Release
```

Use `-DCMAKE_BUILD_TYPE=Release` (debug builds are unsupported by this skill). The `cd` must be in the same command line as `cmake`; the agent's `bash` tool does not preserve `cwd` between separate calls.

Scan the output for any of these failure indicators and abort on match:

| Pattern in output | Abort message |
| --- | --- |
| `source directory` followed by `does not appear to contain CMakeLists.txt` | "The `cd src/llama.cpp` did not take effect. The bash tool may have reset the working directory. Re-run with `cd` in the same command line as `cmake`." |
| `Could NOT find CUDA` | "CMake did not find CUDA. Verify that the toolkit is installed and that `nvcc` is on `PATH`." |
| `CMAKE_CUDA_ARCHITECTURES` followed by `is not one of the following` | "CMake rejected `CMAKE_CUDA_ARCHITECTURES`. The value must be a semicolon-separated list of integers (e.g. `120;89`), not the `X.Y` form from `nvidia-smi`. Re-check step B." |
| `CUDA error` | "CMake configure failed with a CUDA error. Read the lines above for the specific cause." |
| `No CUDA arch specified` | "CMake did not detect any CUDA architecture. Pass `-DCMAKE_CUDA_ARCHITECTURES` explicitly with the value from step B." |

If the configure finishes without those errors, capture the line that starts with `-- Configuring done` as the success marker.

## E. Build

`<N>` is the value of `nproc` captured in step A. The `cd` must be in the same command line as `cmake` (see step D).

**Run the build in the background.** This is the prescribed method, not a workaround. The build duration depends on the host's CPU core count, GPU count, and CUDA architecture count, none of which the skill can know in advance. The agent must not block its own context window on a build of unknown duration. Set the bash tool's timeout long enough for the launch to return control (a few seconds), not for the build to finish.

`-j <N>` is mandatory. Without it, the build will appear to hang on any machine with more than one CPU core.

```bash
cd src/llama.cpp && nohup cmake --build build --config Release -j <N> > /tmp/llama-build.log 2>&1 &
echo "BUILD_PID=$!"
```

Poll the log and the process list on a fixed interval (60-120 seconds is a reasonable default; do not poll more often than every 30 seconds or the agent wastes turns):

```bash
sleep 90 && tail -3 /tmp/llama-build.log && echo "---" && \
  ps -ef | grep -E "cmake|cc1plus|nvcc" | grep -v grep | wc -l && echo "active build procs" && \
  ls src/llama.cpp/build/bin/ 2>/dev/null | wc -l && echo "binaries so far"
```

The build is complete when **all** of these hold:

- The log shows the equivalent of `[100%] Built target llama-server` (the actual final line depends on the cmake generator and parallelism).
- The process list shows no active `nvcc` / `cc1plus` / `cmake` processes.
- `ls src/llama.cpp/build/bin/` shows the three expected binaries (`llama-server`, `llama-cli`, `llama-completion`).

The agent must wait for the build to finish before continuing to step F. If the build exits non-zero, abort: "Build failed. Read the last 50 lines of output for the failing target and report the error to the user. Do not retry automatically."

## F. Verify the binaries

```bash
cd src/llama.cpp
test -x build/bin/llama-server     || { echo "missing: build/bin/llama-server"; exit 1; }
test -x build/bin/llama-cli        || { echo "missing: build/bin/llama-cli"; exit 1; }
test -x build/bin/llama-completion || { echo "missing: build/bin/llama-completion"; exit 1; }

build/bin/llama-server --version
build/bin/llama-cli --version
```

`--version` prints `version: 1 (<short commit SHA>)`. Verify the build is from the pinned source by comparing the short SHA in the binary's output against the tag's peeled commit (use `^{commit}` to skip the tag-object SHA on annotated tags):

```bash
TAG_SHA="$(git rev-parse --short 'refs/tags/gguf-v0.19.0^{commit}')"
SERVER_SHA="$(build/bin/llama-server --version 2>&1 | grep -Eo '\([0-9a-f]+\)' | tr -d '()' | head -1)"
[ "$SERVER_SHA" = "$TAG_SHA" ] || { echo "llama-server commit $SERVER_SHA does not match tag gguf-v0.19.0 ($TAG_SHA)"; exit 1; }
```

If mismatched, abort: "Binary commit does not match the pinned tag `gguf-v0.19.0` ($TAG_SHA). Re-do steps C and D from a clean clone."

`llama-cli` and `llama-completion` come from the same build, so no separate SHA check is needed for them.

## G. Stage the build context

```bash
rm -rf build-context && mkdir -p build-context/bin
cp src/llama.cpp/build/bin/llama-server      build-context/bin/
cp src/llama.cpp/build/bin/llama-cli         build-context/bin/
cp src/llama.cpp/build/bin/llama-completion  build-context/bin/
cp src/llama.cpp/build/bin/*.so*             build-context/bin/
```

The Dockerfile's `COPY bin/*.so*` mirrors the `*.so*` glob above; if both copy the same set, the three expected SONAMEs (libggml.so.0, libllama.so.0, libmtmd.so.0) will be present in the image.

Verify the expected files are present:

```bash
ls build-context/bin/llama-server build-context/bin/llama-cli build-context/bin/llama-completion \
  build-context/bin/libggml.so.0 build-context/bin/libllama.so.0 build-context/bin/libmtmd.so.0
```

If any are missing, abort: "Build context is incomplete; the host build at `src/llama.cpp/build/bin/` is missing one of the expected files. Re-run steps A–F."

## H. Build the Docker image

The Dockerfile lives at `references/compile/Dockerfile` (sibling to this file). The image tag encodes the source pin.

`docker build -f` resolves the path against the build context, not the cwd, so a relative path fails with `lstat references: no such file or directory`. Use an absolute path:

```bash
docker build \
  -f "$(pwd)/references/compile/Dockerfile" \
  -t llama-wizard-llama-cpp:gguf-v0.19.0 \
  build-context
```

## I. Verify the image

The `--version` flag calls `ggml_cuda_init` to enumerate visible devices (even though it does not load a model), so a clean run also validates the nvidia container runtime wiring. The commit SHA in the output is the same one step F validated; this step only validates the container side.

```bash
docker run --rm --gpus all llama-wizard-llama-cpp:gguf-v0.19.0 --version
```

Failures:

- **Exit non-zero** (e.g. `docker: error: ... no CUDA devices visible`): the nvidia runtime is not seeing the GPUs. Abort: "`docker run --gpus all` failed; the nvidia container runtime is not exposing the GPUs to containers. Re-run detect.md and validate step F."
- **Exit zero, but `ggml_cuda_init` reports zero devices**: the runtime is registered but not actually mounting the devices. Abort: "`ggml_cuda_init` reported zero devices inside the container. Re-run detect.md and validate step F (nvidia runtime registration)."
- **Exit zero and `ggml_cuda_init` lists the host's GPUs**: success. Proceed to step J.

## J. Report to the user

After successful verification, present:

```
Compile report
--------------
Source tree:    src/llama.cpp/
Pin:            gguf-v0.19.0 (commit resolved at build time via `git rev-parse refs/tags/gguf-v0.19.0^{commit}`)
CUDA toolkit:   Cuda compilation tools, release <major>.<minor>, V<release>   (line of `nvcc --version` that contains "release")
CUDA arch list: <value from step B, in integer form>
CMake:          <output of `cmake --version | head -1`>   (e.g. `cmake version 4.2.3`)
Host binaries:
  src/llama.cpp/build/bin/llama-server     <size>   <sha256 prefix>
  src/llama.cpp/build/bin/llama-cli        <size>   <sha256 prefix>
  src/llama.cpp/build/bin/llama-completion <size>   <sha256 prefix>
Docker image:   llama-wizard-llama-cpp:gguf-v0.19.0
Image id:       <output of `docker images --no-trunc --quiet llama-wizard-llama-cpp:gguf-v0.19.0` (first 12 chars)>
Image size:     <output of `docker images llama-wizard-llama-cpp:gguf-v0.19.0 --format "{{.Size}}"`>
Smoke test:     `docker run --rm --gpus all llama-wizard-llama-cpp:gguf-v0.19.0 --version` → `version: 1 (a290ce626)`

Compile verdict: ready
```

Generate the size and sha256 lines with:

```bash
ls -l src/llama.cpp/build/bin/llama-server src/llama.cpp/build/bin/llama-cli src/llama.cpp/build/bin/llama-completion | awk '{print $5, $NF}'
sha256sum src/llama.cpp/build/bin/llama-server src/llama.cpp/build/bin/llama-cli src/llama.cpp/build/bin/llama-completion | awk '{print $1, $NF}'
```

The "Compile verdict" here is the compile step's verdict, independent of `detect.md`'s verdict: "the binaries exist, were built from the pinned source tree, and were packaged into a Docker image that the runtime can use." It does **not** mean the server is running or that any model is loaded. If any verification in steps A–I failed and the flow aborted, do not print this report.

## What this file does NOT do

- It does not download models. (See `references/models/` — forthcoming.)
- It does not start the server with a real model loaded. That is `references/compose.md` (forthcoming).
- It does not install system packages. The user must do that and re-invoke the skill from the top.
