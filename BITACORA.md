# BITACORA — llama-wizard

> Operational log of the skill. Each entry is a step executed **manually** with its real output.
> Recipes in `recipes/<DISTRO>/` are written from this log, not the other way around.
> Entry format: timestamp, command, summarised output, result, notes/decisions.

## Conventions

- **Command**: the exact command that was run, copy-pasteable.
- **Output**: the relevant part. If too long, summarise and point to the capture file.
- **Result**: ✅ OK / ⚠️ WARN / ❌ FAIL.
- **Notes/Decisions**: what we learned on this step and how it affects the recipes.

## Session index

| # | Date | Flow step | Result | Notes |
| --- | --- | --- | --- | --- |
| 01 | 2026-06-07 | Environment detection | ✅ OK | First run on Ubuntu 26.04 WSL2. 2× NVIDIA GPUs detected. |
| 02 | 2026-06-07 | Subagent test of detect.md (6 iterations) | ✅ Robust | 18 improvements applied, 1 rejection maintained (per-distro table), resolved with a hybrid heuristic+fallback. |

---

## Session 01 — 2026-06-07 — Environment detection

**Goal**: confirm Linux distro, kernel, WSL flag, CPU, RAM, GPU, drivers, Docker, disk, and tooling on the author's machine. This is the input for `detect.md` and `requirements.md`.

**Command**: a single block that probes every relevant signal. See full capture at the bottom.

### 1.1 OS and kernel

**Command**:
```bash
cat /etc/os-release
uname -a
uname -m
```

**Result**: ✅ OK
```
PRETTY_NAME="Ubuntu 26.04 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04 (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
Linux DESKTOP-55N8ULR 6.6.114.1-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Mon Dec  1 20:46:23 UTC 2025 x86_64 GNU/Linux
arch: x86_64
```

**Notes/Decisions**:
- Distro is `ubuntu-26.04`, codename `resolute`. Very recent, pre-release at the time of writing. Recipe folder will be `recipes/ubuntu-26.04-wsl2/`.
- Kernel is the WSL2 kernel (`microsoft-standard-WSL2`), confirming WSL2 environment.

### 1.2 WSL flag

**Command**:
```bash
grep -Ei "microsoft|wsl" /proc/version
ls -la /run/WSL 2>/dev/null
echo "WSL_INTEROP=${WSL_INTEROP:-unset}"
echo "WSL_DISTRO_NAME=${WSL_DISTRO_NAME:-unset}"
```

**Result**: ✅ OK
```
Linux version 6.6.114.1-microsoft-standard-WSL2 (...) #1 SMP PREEMPT_DYNAMIC ...
/run/WSL contains 1932_interop, 1_interop, 2007_interop, 20772_interop, 20781_interop, 2407_interop, 2_interop, 341_interop (sockets and symlinks)
WSL_INTEROP=/run/WSL/2407_interop
WSL_DISTRO_NAME=Ubuntu
```

**Notes/Decisions**:
- WSL2 confirmed three ways: kernel name, `/run/WSL` directory, and `WSL_INTEROP` env var. Recipe must use `WSL2` in its name.
- WSL2 implies no `systemd` by default. Anything that needs systemd (e.g. `systemctl start docker`) will fail. Docker is running because WSL2 starts it differently (or it was started by Docker Desktop on the host).

### 1.3 CPU and RAM

**Command**:
```bash
lscpu | grep -E "Model name|^CPU\(s\)|Thread|Socket|Core"
free -h
```

**Result**: ✅ OK
```
CPU(s):                                  16
Model name:                              AMD Ryzen 7 7800X3D 8-Core Processor
Thread(s) per core:                      2
Core(s) per socket:                      8
Socket(s):                               1
Mem:            46Gi       1.9Gi        44Gi        11Mi       565Mi        44Gi
Swap:           12Gi          0B        12Gi
```

**Notes/Decisions**:
- AMD Ryzen 7 7800X3D: 8 physical cores, 16 threads. Plenty for llama.cpp CPU offload.
- 46 GiB RAM. Model loading will fit easily for any model < 32 GB (FP16) or much larger if quantised.
- 12 GiB swap. Fine.

### 1.4 GPU

**Command**:
```bash
lspci | grep -Ei "VGA|3D|Display"
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv,noheader
rocm-smi --showproductname  # if installed
```

**Result**: ✅ OK
```
lspci: no match (lspci appears unavailable in this WSL2)
nvidia-smi:
  NVIDIA GeForce RTX 5090, 596.36, 32607 MiB
  NVIDIA GeForce RTX 4080 SUPER, 596.36, 16376 MiB
rocm-smi: NOT INSTALLED
```

**Notes/Decisions**:
- **`lspci` does not work in this WSL2 instance.** The PCI device tree is not exposed. Recipe must use `nvidia-smi` as the **primary** GPU detection source on WSL2.
- **2× NVIDIA GPUs detected**:
  - RTX 5090 (Blackwell), 32607 MiB (~31.8 GiB) VRAM, driver 596.36.
  - RTX 4080 SUPER (Ada Lovelace), 16376 MiB (~16.0 GiB) VRAM, driver 596.36.
  - Combined: ~48 GiB VRAM.
- Driver 596.36 is recent enough for RTX 5090 (Blackwell needs >= 570.x typically). Looks good.
- AMD ROCm not installed. Fine, not needed since we have NVIDIA.
- **Impact on model catalog**: with 48 GiB combined VRAM, the catalog can include 70B-class models with Q4 quantisation (~40 GB), 32B at high precision, and small models for fast iteration. This is a much wider catalog than we originally planned.

### 1.5 Docker

**Command**:
```bash
docker --version
docker info 2>&1 | head -n 15
docker info 2>&1 | grep -i "runtimes\|nvidia"
```

**Result**: ✅ OK
```
Docker version 29.5.2, build 79eb04c
... plugins: agent, ai, buildx, compose ...
Runtimes: io.containerd.runc.v2 nvidia runc
```

**Notes/Decisions**:
- Docker 29.5.2, daemon responding.
- **The `nvidia` runtime is already registered in Docker.** This means the container can request GPU access via `docker run --gpus ...` or `deploy.resources.reservations.devices` in compose. The setup is already correct.
- We did not need `nvidia-ctk` as a binary because the runtime was pre-configured (likely by Docker Desktop or a manual setup). The `nvidia-container-toolkit` packages are likely installed as a dependency even if `nvidia-ctk` is not on PATH.

### 1.6 Disk space

**Command**:
```bash
df -h .
```

**Result**: ✅ OK
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdd       1007G  4.1G  952G   1% /
```

**Notes/Decisions**:
- 952 GiB free on the working filesystem. The repo lives on `/home/jorge/llama-wizard/` which is on `/dev/sdd`. `models/` and `vendor/` will live here too. No space concerns; can hold multiple large models.

### 1.7 Tooling

**Command**:
```bash
for tool in git make cmake gcc g++ curl jq pkg-config sudo; do
  command -v "$tool" && "$tool" --version | head -n 1 || echo "$tool: MISSING"
done
```

**Result**: ⚠️ WARN
```
git:        OK   git version 2.53.0
make:       MISSING
cmake:      MISSING
gcc:        OK   gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0
g++:        OK   g++ (Ubuntu 15.2.0-16ubuntu1) 15.2.0
curl:       OK   curl 8.18.0
jq:         MISSING
pkg-config: OK   2.5.1
sudo:       OK   sudo-rs 0.2.13-0ubuntu1
```

**Notes/Decisions**:
- **`make`, `cmake`, and `jq` are missing.** These are blockers for the next step (`requirements.md` validation, then `compile/llama-cpp.md`).
- `gcc` and `g++` are present (Ubuntu ships them), which is unusual — usually `build-essential` brings all of them. Maybe `gcc` and `g++` were installed explicitly, or `make` was somehow removed. The fix is to install `build-essential` (which pulls `make` and `cmake`).
- `sudo` is `sudo-rs` (Rust rewrite), a common default on Ubuntu 26.04.

### 1.8 User, groups, and sudo

**Command**:
```bash
id -nG
sudo -n true && echo "sudo passwordless OK" || echo "sudo requires password"
```

**Result**: ✅ OK
```
user: jorge
groups: jorge adm cdrom sudo dip plugdev users docker
sudo: requires password
```

**Notes/Decisions**:
- User `jorge` is in the `docker` group: no `sudo` needed for `docker` commands.
- `sudo` is **not passwordless**. Any `apt install` step will prompt for a password. Recipe must either accept the prompt or use `sudo -S` with stdin or pre-authorise with `sudo -v` first.

### 1.9 Summary

| Item | Status | Value |
|---|---|---|
| OS | ✅ | Ubuntu 26.04 LTS (resolute) |
| Kernel | ✅ | 6.6.114.1-microsoft-standard-WSL2 (WSL2) |
| CPU | ✅ | AMD Ryzen 7 7800X3D, 8c/16t |
| RAM | ✅ | 46 GiB |
| GPU 0 | ✅ | NVIDIA RTX 5090, 32607 MiB, driver 596.36 |
| GPU 1 | ✅ | NVIDIA RTX 4080 SUPER, 16376 MiB, driver 596.36 |
| Docker | ✅ | 29.5.2, daemon up, `nvidia` runtime registered |
| Disk | ✅ | 952 GiB free on `/dev/sdd` |
| Group docker | ✅ | user `jorge` is in `docker` |
| Tools | ⚠️ | `make`, `cmake`, `jq` MISSING; `git`, `gcc`, `g++`, `curl`, `pkg-config`, `sudo` present |
| sudo | ✅ (with password) | not passwordless |

**Blockers before compile step**:
- Install `make`, `cmake`, `jq`. (One `apt install` line.)

**Impact on recipes**:
- `detect.md` for `ubuntu-26.04-wsl2/` must use `nvidia-smi` as the primary GPU probe (`lspci` does not work in WSL2).
- `requirements.md` for `ubuntu-26.04-wsl2/` must install `make cmake jq` (and probably `build-essential` for safety).
- `compile/llama-cpp.md` can rely on `nvidia-smi` showing GPUs to decide CUDA backend on/off.
- `docker/llama-cpp.md` can use the `nvidia` runtime directly (`docker run --gpus all` or compose `deploy.resources.reservations.devices`).
- `models/CATALOG.md` can include 70B-class models (Q4 fits in 48 GiB combined VRAM).
- The 2 GPUs mean `compose.md` can target `--gpus all` and let llama.cpp pick the split, OR target a single GPU (RTX 5090, 32 GiB) for simplicity. Decision deferred to the compose step.

---

## Next steps (deferred)

- **Next**: install the missing tools (`make`, `cmake`, `jq`) with `sudo apt install -y build-essential cmake jq`. Record in BITACORA session 02.
- **Then**: write `recipes/ubuntu-26.04-wsl2/detect.md` (declarative, in English) based on the commands that worked here.
- **Then**: write `recipes/ubuntu-26.04-wsl2/requirements.md` with the install + validation steps.
- **Then**: write `recipes/ubuntu-26.04-wsl2/compile/llama-cpp.md` and try a real `llama.cpp` clone + build.

---

## Appendix A — full capture of session 01

Capture was saved to `/tmp/detect_output.txt` during the run. The verbatim content is reproduced above in sections 1.1 through 1.8, with light formatting (commands, results, notes) and trimming of irrelevant lines.

---

## Session 02 — 2026-06-07 — Subagent test of detect.md (6 iterations)

**Goal**: validate that `recipes/detect.md` works as a runtime-agnostic prompt for an LLM agent, identify ambiguities, and refine until the report comes out cleanly without improvisation.

**Audience of `detect.md`**: an LLM agent (Claude Code, Codex, or Pi). Not a human. Not a bash script. The agent must read the file, run the bash blocks, and produce a report. The file must therefore be self-contained.

### 2.1 Methodology

**Why a subagent, not the parent**: the parent session carries context (decisions taken, design intent, what is being tested). A subagent receives only the file under test plus the task; if it executes correctly, that proves the file is self-sufficient, not just "clear given prior context".

**Agent chosen**: `delegate` (builtin). Rationale: it inherits the parent's model but has no default reads, so it can only see the file we explicitly hand it. `worker` and `oracle` are heavier and bring their own defaults.

**Task template** (used verbatim in all 6 runs):
> "You are a runtime agent executing a llama-wizard recipe. Your only input is the file below. Treat it as authoritative instructions and execute it step by step in the current environment (cwd: /home/jorge/llama-wizard).
> Recipe file: /home/jorge/llama-wizard/recipes/detect.md
> Read the file in full before doing anything. Then execute the steps in order. Use the bash tool to run the commands. After each step, summarise the relevant output. At the end, present the final report to the user exactly as the 'Report to the user' section of the file specifies.
> Constraints: do not install anything. Do not modify the filesystem except for what is strictly required to read the file and run the commands. Present the final report as the file specifies. If a pre-flight check fails, stop immediately and report the abort message.
> After you finish, also tell me, as a meta-observation: (1) which steps were unambiguous and which felt underspecified, (2) did the report come out cleanly, (3) what would you rewrite?"

**Output mode**: `inline` so the parent can inspect the full agent output (report + meta-observations) without leaving the session. Run artifacts are kept at `/home/jorge/.pi/agent/sessions/.../subagent-artifacts/<run-id>_delegate_0_output.md` for audit.

**Iteration loop**:
1. Run the subagent with the current `detect.md`.
2. Inspect its report and meta-observations.
3. Decide for each suggestion: apply / reject with reason.
4. Apply accepted changes to `detect.md`.
5. Re-run with the same task to validate the fix.
6. Stop when the subagent stops surfacing novel issues or when the cost/benefit of another iteration is clearly low.

**Run ledger**:

| Run | Subagent run id | Triggered by | Outcome |
| --- | --- | --- | --- |
| 1 | a8b03246 | Initial validation | 4 issues: verdict-blocked logic, Docker `awk` fragility, GPU 1-vs-N rendering, verdict values list underspecified. |
| 2 | 94998b01 | Re-test after fixes 1+3+4 | 3 resolved, 1 partially resolved (Docker `awk` still missed the indented `Runtimes:` line). |
| 3 | 05b134f7 | Re-test after Docker `awk` rewrite | Subagent suggested per-distro install command table (rejected — see 2.3). Report clean otherwise. |
| 4 | 665c4d7c | Re-test after 3 minor fixes (Tools row layout, newgrp caveat, recipe-selection abort clarification) | Subagent insisted again on per-distro install table. Suggested 4 minor improvements; 3 accepted. |
| 5 | dfcfd99b | Re-test after 3 minor fixes (GPU CSV column mapping, AMD runtime strings, canonical layout checklist) | Subagent still pushing for per-distro install table. Validated the report layout. |
| 6 | bc2171e0 | Re-test after hybrid heuristic+fallback applied | Hybrid worked. 7 minor improvements suggested; 6 accepted as the final round (NUMA / busybox `free` rejected as out-of-scope for the target distros). |

### 2.2 Issue / fix ledger (consolidated)

| # | Origin run | Issue | Action | Reason |
| --- | --- | --- | --- | --- |
| 1 | 1 | Verdict-blocked logic for missing tools was implicit; subagent had to invent that case | Applied | Concrete gap in the contract |
| 2 | 1, 2 | Docker `awk '/^Runtimes:/'` missed the indented line on Docker 29.5.2 | Applied | Fragility, real breakage on this host |
| 3 | 1 | GPU table rendering for 1 vs N GPUs not specified | Applied | Concrete gap |
| 4 | 1 | Verdict values list didn't enumerate cases | Applied | Concrete gap |
| 5 | 3 | Per-distro install command table (apt/dnf/pacman mapping) | **Rejected** | User intent: keep install declarative, let the LLM infer |
| 6 | 3 | rocm-smi JSON key path for AMD rendering | Applied | Concrete gap (less common path) |
| 7 | 3 | AMD runtime string in Docker `Runtimes:` | Applied | Concrete gap |
| 8 | 3 | Canonical layout checklist for `blocked`-for-tools verdict | Applied | Reinforce an existing prose description |
| 9 | 3 | `Tools:` row render rule should be a labelled spec, not a nested bullet | Applied | Easy to lose in the bullet |
| 10 | 3 | `test -d` step after recipe selection | Applied | Strengthen existing prose |
| 11 | 4 | `Selected recipe:` placement explicit in `blocked` case | Applied | Subagent was making a judgment call |
| 12 | 4 | `newgrp docker` caveat (only affects current shell) | Applied | Real footgun |
| 13 | 4 | Recipe-selection "abort" rows: table before or after the abort message | Applied | Clarification |
| 14 | 4 | "No recipe" verdict semantics | Applied | Clarification |
| 15 | 4 | `df` working directory not pinned | Applied | Pin to `$HOME` for cwd independence |
| 16 | 5 | GPU CSV column order from `nvidia-smi` | Applied | Spell out explicitly |
| 17 | 6 | NUMA / busybox `free` edge cases | **Rejected** | Out of scope for the target distros; re-evaluate if the skill grows |
| 18 | 6 | WSL matching algorithm in recipe-selection table | Applied | Ambiguity in the matching rule |
| 19 | 6 | `id -un` for the Docker-group user label | Applied | Subagent was copying the example literally |
| 20 | 6 | CPU model string rendered verbatim | Applied | Avoid improvisation |
| 21 | 6 | `Verdict: ready with warnings` conditions enumerated | Applied | Clarification |
| 22 | 6 | `; exit` in `lscpu` awk to avoid NUMA repetition | Applied | Determinism |

Total: 18 applied, 4 rejected (3 NUMA/busybox, 1 per-distro table). One of the rejected items (per-distro table) was subsequently revisited and resolved via the hybrid below.

### 2.3 The hybrid (declarative install + fallback) — why

The subagent pushed for a per-distro install command table in 5 out of 6 runs. The user's intent (twice confirmed) is to keep install inference in the LLM, not in a hardcoded table. The two are not strictly opposed, so a hybrid was designed:

- The LLM applies a small heuristic (`ubuntu`/`debian` → `apt`, `fedora`/`rhel` → `dnf`, `arch`/`manjaro` → `pacman`, `alpine` → `apk`, `opensuse*` → `zypper`).
- If the heuristic does not match, the agent aborts with a clear "no install command registered" message and points the user to the heuristic list to extend it.

This preserves the declarative surface (no per-distro install table to maintain) while adding a deterministic decision procedure (the heuristic) and a safety net (the fallback message) for uncommon distros.

The hybrid was tested in run 6; the subagent successfully inferred `apt` for `ubuntu` and produced the expected `sudo apt install -y make cmake jq` command.

### 2.4 Why we stopped at 6 runs

- All 3 original issues (verdict-blocked logic, Docker `awk`, GPU rendering) are resolved.
- 18 of 22 subagent suggestions have been applied or explicitly rejected with reason.
- The remaining suggestions in run 6 are either edge cases out of scope (NUMA, busybox) or stylistic micro-improvements whose cost/benefit is unclear.
- The report comes out cleanly across all 6 runs.
- Further iterations risk "polishing the chrome" without changing real behaviour. We stop here and move to the next recipe.

### 2.5 Final state of `recipes/detect.md`

- 240 lines.
- Distro-agnostic. Sits at `recipes/detect.md`, not under any `recipes/<DISTRO>/`.
- Self-guard: nothing to install from this file.
- All detection steps (A–I) have been validated on Ubuntu 26.04 WSL2 by 6 independent subagent runs.
- Ready to be the first file the skill entry point loads, before any distro-specific recipe.

### 2.6 Next concrete step (session 03 plan)

Clone `llama.cpp` manually into `vendor/llama.cpp`, capture the build output (branch, commit SHA, CMake configuration flags that worked on this host, time to compile, final binary path), and use that as input for `recipes/ubuntu-26.04-wsl2/compile/llama-cpp.md`.
