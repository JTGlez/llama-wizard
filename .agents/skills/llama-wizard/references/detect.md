# Environment detection

## Goal

Detect the host environment, gate the skill on a verdict (ready / ready with warnings / blocked), and present a structured report to the user before any install or compile step. The detected distro is reported but does not select a flow folder; flows under `references/` are distro-agnostic.

## Scope

This file is distro-agnostic. It runs first, before any flow is loaded. There are no per-distro instruction folders; all flows under `references/` are distro-agnostic.

## Pre-flight (abort on failure)

### 1. Must be Linux

```bash
uname -s
```

Expected: `Linux`. Anything else → abort with: "llama-wizard requires Linux. Detected: <uname output>."

### 2. Architecture must be x86_64

```bash
uname -m
```

Expected: `x86_64`. Other values → abort with: "llama-wizard is for x86_64 only. Detected: <arch>."

## Detection steps

Run in order. Capture the full output for the report.

### A. Distro identification

```bash
. /etc/os-release
printf 'id=%s\nid_like=%s\nversion_id=%s\ncodename=%s\npretty=%s\n' \
  "$ID" "${ID_LIKE:-}" "$VERSION_ID" "${VERSION_CODENAME:-}" "${PRETTY_NAME:-}"
```

### B. WSL flag

```bash
grep -Ei "microsoft|wsl" /proc/version >/dev/null && echo "wsl=true" || echo "wsl=false"
echo "WSL_INTEROP=${WSL_INTEROP:-unset}"
echo "WSL_DISTRO_NAME=${WSL_DISTRO_NAME:-unset}"
```

### C. Kernel

```bash
uname -r
```

### D. CPU and RAM

```bash
lscpu | awk -F: '/Model name/ {gsub(/^ +/,"",$2); print "cpu_model="$2; exit}'
lscpu | awk -F: '/^CPU\(s\):/ {gsub(/^ +/,"",$2); print "cpu_threads="$2; exit}'
free -h | awk '/^Mem:/ {print "ram_total="$2, "ram_available="$7}'
```

Print the `Model name` and `CPU(s)` fields verbatim from `lscpu`; do not strip vendor-specific suffixes (e.g. "8-Core Processor"). The `; exit` guards against NUMA topologies that repeat these lines.

### E. GPU detection

Probe NVIDIA first, then AMD.

```bash
if command -v nvidia-smi >/dev/null 2>&1; then
  nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv,noheader,nounits 2>/dev/null
  echo "gpu_vendor=nvidia"
elif command -v rocm-smi >/dev/null 2>&1; then
  rocm-smi --showproductname --json 2>/dev/null
  echo "gpu_vendor=amd"
else
  echo "gpu_vendor=none"
  echo "gpu_reason=no nvidia-smi and no rocm-smi"
fi
```

`lspci` is intentionally not used; it is unreliable on WSL2 and other virtualised hosts.

If `gpu_vendor=none` → warn: "No discrete GPU detected. The server will run on CPU only. Smaller models will be slow." Continue unless the user aborts.

When rendering the report, list one row per GPU returned by `nvidia-smi` or `rocm-smi`. The exact column order from `nvidia-smi --query-gpu=index,name,driver_version,memory.total` is `[index, name, driver, memory.total]`. Render each GPU as `GPU <index>: <name>  (<memory.total> MiB, driver <driver>)`. If `nvidia-smi` reports zero rows (driver present but no GPU), fall through to `rocm-smi`. For AMD, `rocm-smi --showproductname` returns the marketing name only; if `--json` is used, the GPU name is under `.card0.Product Name` and driver under `.card0.Driver Version`. Render AMD GPUs as `GPU 0: <name>  (driver <driver>)` without the memory column. If neither reports any GPU, the row collapses to "GPU: none detected (CPU-only mode)".

### F. Docker engine

```bash
docker --version 2>/dev/null
docker info 2>/dev/null > /tmp/docker_info.txt
awk '/^[^A-Za-z]*Server Version/ || /^[^A-Za-z]*Runtimes:/' /tmp/docker_info.txt
rm -f /tmp/docker_info.txt
```

If `docker` is missing or `docker info` fails → abort with: "Docker is required. Install Docker Engine for your distro and ensure the daemon is running, then re-run this skill."

If a GPU was detected, read the captured `Runtimes:` line and confirm that `nvidia` (NVIDIA) or one of `rocm`, `amd`, `runc-amd` (AMD) is present as a runtime. If the expected runtime is absent → abort with: "Docker is installed but the GPU runtime is not registered. Install nvidia-container-toolkit (NVIDIA) or the AMD equivalent and register the runtime, then re-run."

The `awk` pattern matches both `Server Version:` and `Runtimes:` regardless of leading indent (Docker 29.5.2 prints them as ` Server Version:` and ` Runtimes:` under the `Server:` block, while some versions print them at column 0). It is more robust than strict `^Server Version` or `^Runtimes:` patterns.

### G. Disk space

The cwd of the agent must be the repo root. Use an absolute path so the check is independent of the agent's working directory.

```bash
df -BG "$HOME" | awk 'NR==2 {gsub(/G/,"",$4); print "free_gib="$4}'
```

If `free_gib < 20` → abort with: "Less than 20 GiB free. llama-wizard needs at least 20 GiB to compile llama.cpp and host a model."

If `free_gib < 50` → warn but continue: "Less than 50 GiB free. Small models will fit; larger ones may not."

### H. Build and runtime tools

Split the tool check into two passes: **required** (without which the skill itself cannot run) and **optional** (needed only for the build-from-source path).

```bash
for tool in git make cmake gcc g++ curl jq pkg-config; do
  if command -v "$tool" >/dev/null 2>&1; then
    v=$("$tool" --version 2>/dev/null | head -n 1)
    printf 'tool=%s status=ok version="%s"\n' "$tool" "$v"
  else
    printf 'tool=%s status=missing\n' "$tool"
  fi
done
command -v nvcc >/dev/null 2>&1 && echo "nvcc_present=true" || echo "nvcc_present=false"
```

**Required tools** (must be present or the verdict is `blocked`):
`git`, `make`, `cmake`, `gcc`, `g++`, `curl`, `jq`, `pkg-config`. Their roles:
- `git` — clone llama.cpp source
- `make`, `gcc`, `g++` — build llama.cpp (these typically come together in a distro's "build" or "base-devel" group)
- `cmake` — generate the llama.cpp build files
- `curl` — download model files
- `jq` — parse JSON outputs during validation
- `pkg-config` — locate system libraries for llama.cpp

If any required tool is missing:

- Set the verdict to `blocked`. Do not proceed to the flow load.
- List the missing tools and their role from the list above.
- Infer the install command for the detected distro using this heuristic:
  - `id=ubuntu`, `id=debian`, or `id_like=debian` → `apt`
  - `id=fedora`, `id=rhel`, `id=centos`, or `id_like=rhel` → `dnf`
  - `id=arch`, `id=manjaro`, or `id_like=arch` → `pacman`
  - `id=alpine` → `apk`
  - `id=opensuse*` → `zypper`
  - Anything else → use the next step (fallback).
- Fallback: if no heuristic matches, do **not** guess. Abort the verdict with: "Detected distro `<id> <version>` has no install command registered in detect.md. Contribute the install command to detect.md's heuristic list." Print the missing tools and the heuristic list so the contributor can copy the format.
- The user runs the install manually, then re-invokes the skill so this flow runs again from the top and validates.
- **Render rule for the `Tools:` row**: render missing tools inline in a single row, e.g. `Tools: git OK, make MISSING, cmake MISSING, gcc OK, g++ OK, curl OK, jq MISSING, pkg-config OK`. Do not split into two rows. Include the `nvcc` check as an extra note in the row when relevant, e.g. `Tools: ... pkg-config OK  (nvcc: not installed)`.
- Canonical layout when the verdict is `blocked` due to missing tools (checklist; use this exact order):
  1. The report table (the same `Environment report` block as the other verdicts), including the `Selected flow:` line.
  2. A "Missing build tools" section, one bullet per missing tool, each with the tool's role from the list above.
  3. The inferred install command in a single bash code block (or, in the fallback case, the "no install command registered" message).
  4. The verdict line.

**Optional tool**:
- `nvcc` (NVIDIA CUDA Toolkit compiler). Required only for the build-from-source path. **Missing `nvcc` is a warning, not a block.** If `nvcc_present=false`:
  - The verdict is `Verdict: ready` (NOT `ready with warnings` — that verdict is reserved for disk/GPU conditions, see the Verdict values section). The only thing that changes is the addition of a `Warnings` block.
  - Place the `Warnings` section between the report table and the `Selected flow:` line.
  - The `Warnings` block must include both the explanation and a hint pointing the user to the official install instructions. The exact text to render is:
    ```
    Warnings

    - `nvcc` is not installed. Only required if you compile `llama.cpp` from source; the pre-built image `ghcr.io/ggml-org/llama.cpp:server-cuda` does not need it.
    - To install the CUDA Toolkit, follow the official instructions at https://developer.nvidia.com/cuda-downloads (the page lets you pick your distro and version). The user must run the install manually; re-invoke the skill afterwards.
    ```
  - Do not give distro-specific install commands or .run installer scripts. The official page handles distro/version selection. The user is responsible for picking the right combo.
  - If the GPU is NVIDIA and the user picks the build-from-source path later, the compile flow will abort and tell the user the same hint (linking to the same page) so the message is consistent across flows.

### I. User in docker group

```bash
id -nG | tr ' ' '\n' | grep -qx docker && echo "in_docker_group=true" || echo "in_docker_group=false"
CURRENT_USER=$(id -un 2>/dev/null || echo unknown)
```

If `in_docker_group=false` and a GPU is present → tell the user:

```
sudo usermod -aG docker "$USER"
newgrp docker
```

Note: `newgrp docker` only affects the current shell session. After the user re-invokes the skill, the new group membership must be picked up by the launching shell (typically a new login shell or a fully restarted terminal). Then ask them to re-run the skill.

In the report, render the row as `Docker group: user <CURRENT_USER> is in docker group` (or `is not in docker group` if absent). Always derive the label from `id -un`; do not copy the example literally.

## Flow selection

Flows under `references/` are distro-agnostic: their commands do not change with the host's distro. The only step that ever needed distro-specific knowledge was this detection step (which is now done). The skill therefore does not select a "flow folder"; the next step (driven by the skill entry point) loads the path-gate flows (`compile/`, `compose`, `models/`) directly.

The role of the detection result is to **report** what was found (distro, kernel, WSL flag, GPUs, tools) and to **gate** the next step on the verdict. The verdict is computed in the next section.

## Report to the user

Present one table, then the selected flow, then a verdict. Example shape:

```
Environment report
------------------
OS:               Ubuntu 26.04 LTS (resolute)
Kernel:           6.6.114.1-microsoft-standard-WSL2  (WSL2: yes)
Architecture:     x86_64
CPU:              AMD Ryzen 7 7800X3D  (16 threads)
RAM:              46 GiB total, 44 GiB available
GPU 0:            NVIDIA RTX 5090  (32607 MiB, driver 596.36)
GPU 1:            NVIDIA RTX 4080 SUPER  (16376 MiB, driver 596.36)
Docker:           29.5.2  (nvidia runtime: registered)
Disk free:        952 GiB
Tools:            git OK, make OK, cmake OK, gcc OK, g++ OK, curl OK, jq OK, pkg-config OK
Docker group:     user jorge is in docker group

Selected flow:  references/compile/llama-cpp.md   (and references/compose.md, forthcoming)

Verdict: ready
```

Verdict values:

- `Verdict: ready` — all checks passed (including all tools from step H). Continue to the next step (load the path gate or whatever the skill entry point says is next).
- `Verdict: ready with warnings` — proceed but call out the warnings above the table. Use this verdict only when **all** of these hold:
  - All pre-flight checks passed.
  - All tools from step H are present.
  - One or more of these non-blocking conditions is true: (a) `20 <= free_gib < 50`, (b) `gpu_vendor=none`.
  Do not use this verdict for missing tools, missing runtimes, or pre-flight failures.
- `Verdict: blocked` — something required is missing or broken. Three cases:
  - **Pre-flight or infrastructure failure** (not Linux, not x86_64, no Docker, no GPU runtime): the user must fix the system before re-running. Print the report table first, then the abort message, then the verdict.
  - **Missing build tools** (step H): the user must install the listed tools, then re-invoke the skill. Print the report table first (including the `Selected flow:` line, so the user knows which flow will be loaded after fixing tools), then a "Missing build tools" section listing each missing tool and its role, then the inferred install command in a bash block, then the verdict. This is the canonical layout for this case.
- Do not use any other verdict.

This mirrors the "missing build tools" case where the report is printed first and the install command is shown after.

## What this file does NOT do

- It does not install anything.
- It does not modify the filesystem.
- It does not write any state file.
- It does not load any flow. The next step (driven by the skill entry point) is responsible for loading the path-gate flows directly from `references/compile/`, `references/compose.md`, `references/models/`, etc.
- It does not download models or clone repos.
