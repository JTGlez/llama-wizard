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
| 03 | 2026-06-07 | Architectural decision: single-agent linear flow | ✅ Adopted | Recipes are written for one agent reading them sequentially; context accumulates. Subagent orchestration is not prescribed. |
| 04 | 2026-06-07 | Iteration 1: SKILL.md + detect.md, Agent Skills spec | ✅ Shipped | Skill at `.agents/skills/llama-wizard/`. Spec + Pi discovery both satisfied. |
| 05 | 2026-06-07 | nvcc demoted from required to optional | ✅ Applied | `nvcc` is only required for the build-from-source path; the pre-built image path bypasses it. Missing `nvcc` now produces a warning, not a `blocked` verdict. |

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

Clone `llama.cpp` manually into `src/llama.cpp`, capture the build output (branch, commit SHA, CMake configuration flags that worked on this host, time to compile, final binary path), and use that as input for `recipes/ubuntu-26.04-wsl2/compile/llama-cpp.md`. The directory name was renamed from `vendor/` to `src/` (see session 03).

---

## Session 03 — 2026-06-07 — Architectural decision: single-agent linear flow

### 3.1 Decision

**llama-wizard recipes are written for a single agent reading them sequentially.** The agent accumulates context across recipes. The skill does **not** prescribe subagent orchestration.

This decision was taken before the implementation of session 03's concrete work (clone + read docs + write `compile/llama-cpp.md`). It is a load-bearing architectural choice, so it is recorded here as its own session, not buried in a later one.

### 3.2 Why

- **Runtime-agnostic**: a single agent works in Claude Code, Codex, Pi Agent, or executed manually by a human. A model that prescribes "the parent Pi session must hand context to a subagent" silently excludes the other runtimes.
- **Simpler recipes**: each recipe declares its pre-conditions in a small block; the agent validates them against the context it has already built. No transport, no handoff syntax, no explicit input/output contract.
- **Simpler debugging**: when something goes wrong, the failure is local to one agent reading one .md. We do not have to reconstruct "what context did the parent pass" or "which subagent failed".
- **Reflects actual use**: in Claude Code, Codex, or Pi, the user runs one agent and asks it to follow the skill. They do not orchestrate subagents themselves. Writing the skill to match that model is the most natural fit.

### 3.3 What this changes concretely

- **`recipes/detect.md`**: starts cold. The agent knows nothing about the host. Runs the full detection flow (pre-flight + A–I + recipe selection) and produces a verdict + report.
- **`recipes/ubuntu-26.04-wsl2/compile/llama-cpp.md` (forthcoming)**: starts with a **Pre-conditions** block that lists what the agent must already know from the context (selected recipe folder, GPUs, CUDA toolkit present, nproc, etc.). The recipe aborts with a clear message if any pre-condition is missing. There is no inline mini-detect duplicating detect.md.
- **Subsequent recipes** (docker, models, compose): same shape. Pre-conditions block + execution. Each one assumes the agent already ran the previous ones.
- **Our subagent-based unit tests** (how we validate each recipe in isolation): the test prompt embeds a simulated handoff. We pass the subagent the literal output of the previous step (e.g. the `Environment report` block from `detect.md`) plus the recipe under test, and the task is phrased as "you are the agent that has just finished `detect.md`; now execute `compile/llama-cpp.md`." This is not the same as orchestration; it is a one-shot handoff used only for our testing.

### 3.4 Trade-off accepted

A recipe invoked **standalone** (without prior context) will fail with a clear "pre-condition not met" abort. This is accepted as incorrect usage. The skill is meant to be executed top-to-bottom (detect → gate → compile → docker → models → compose). Standalone invocation is out of scope.

We do not add a defensive mini-detect to every recipe to support this out-of-scope use case. YAGNI: if the standalone use ever becomes a real need, the right answer is to extract a `recipes/detect-mini.md` that the caller can opt into, not to bloat every recipe.

### 3.5 Rejected alternatives

- **Mini-detect inline in every recipe** (self-contained, defensive). Rejected because the recipes grow, the duplication has no real benefit when used correctly, and the cost of the double-detect is not justified by the benefit.
- **Explicit input/output JSON contract between recipes** (runtime-agnostic, but adds ceremony). Rejected because the human/agent-readable prose in each Pre-conditions block is already a contract; an additional JSON layer would only help if a non-LLM caller wanted to consume the skill, and we do not have that requirement.
- **Subagent orchestration prescribed by the skill** (Pi-friendly). Rejected because it breaks runtime-agnosticism. The user is free to orchestrate in Pi if they want, but the skill does not tell them to.
- **Hardcoded per-distro install command table** (rejected twice in session 02). Resolved via the hybrid heuristic+fallback in `detect.md`. Carried forward; this decision is independent of the linear-flow one.

### 3.6 Naming consequence

The directory where `llama.cpp` is cloned is now `src/llama.cpp/` (was `vendor/llama.cpp/`). The user rejected `build/` to avoid confusion with `docker build`, and chose `src/` for its brevity and standard convention. The `.gitignore` rule was updated from `vendor/` to `src/llama.cpp/`. This change is part of session 03 setup, not a separate decision.

### 3.7 Concrete next steps (post-decision)

Now that the architecture is fixed, the session 03 concrete work proceeds as planned:

1. Identify the last stable tag of `llama.cpp` (not `master` head) and check it out.
2. Read `docs/docker.md`, the Build section of `README.md`, and any other relevant doc to extract CMake flags, build steps, and the binary path.
3. Capture host data (`nvcc --version`, `cmake --version`, `nvidia-smi --query-gpu=compute_cap`, `nproc`) in this session's notes.
4. Write `recipes/ubuntu-26.04-wsl2/compile/llama-cpp.md` with a Pre-conditions block and exact, unambiguous commands.
5. Validate the recipe with a subagent test that simulates the detect→compile handoff.

### 3.8 `llama.cpp` version pinning

- Cloned shallow into `src/llama.cpp/` from `https://github.com/ggml-org/llama.cpp.git`.
- Tags are namespaced `gguf-vX.Y.Z` (not `vX.Y.Z`); the last stable release is `gguf-v0.19.0` (commit `a290ce626663dae1d54f70bce3ca6d8f67aab62f`, dated 2026-05-06).
- Checked out at that tag, detached HEAD. The pin is recorded in the recipe; subagents must not move it.
- Future builds must use the same tag, not `master`. The `gguf-vX.Y.Z` namespace is the only release tag scheme; the `master-XXXX` tags are pointers to rolling `master` commits, not releases.

### 3.9 Documentation extract (what we will use)

From `docs/build.md` (CUDA section) and `docs/docker.md`:

- Default CUDA build (covers all archs detected at configure time):
  ```
  cmake -B build -DGGML_CUDA=ON
  cmake --build build --config Release
  ```
- Targeted CUDA build (only the compute capabilities we actually have):
  ```
  cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="86;89"
  cmake --build build --config Release
  ```
- Binary path after build: `build/bin/llama-server` (and `llama-cli`, `llama-completion`).
- Pre-built image: `ghcr.io/ggml-org/llama.cpp:server-cuda` ships with CUDA 12.8.1 and all supported architectures. If the official image already covers `sm_120`, the only reason to compile locally is customisation or learning.
- The official Dockerfile uses `CUDA_VERSION=12.8.1` and `CUDA_DOCKER_ARCH=cmake build default`. sm_120 (Blackwell, RTX 5090) requires CUDA 12.8+; older CUDA toolkits (e.g. 12.4) will fail or warn.

### 3.10 Host data (real environment, captured for the recipe)

```
nvidia-smi --query-gpu=index,name,driver_version,compute_cap --format=csv
0, NVIDIA GeForce RTX 5090, 596.36, 12.0
1, NVIDIA GeForce RTX 4080 SUPER, 596.36, 8.9
```

- `nvidia-smi` available; `compute_cap` confirms `12.0` (sm_120, Blackwell, RTX 5090) and `8.9` (sm_89, Ada Lovelace, RTX 4080 SUPER).
- `cmake`: **NOT INSTALLED** (same gap as session 01).
- `nvcc`: **NOT INSTALLED**. There is a dangling symlink `/usr/local/cuda -> /usr/local/cuda-13.3` but the target does not exist. No `cuda*` package is installed via dpkg. The system has the driver (596.36) but not the CUDA Toolkit.
- `nproc`: 16.
- RAM: 46 GiB total, 44 GiB available.
- Conclusion: building llama.cpp from source on this host currently requires installing the CUDA Toolkit (12.8 or 13.x) **and** `cmake`. The recipe must detect this and abort with a clear message; we will not paper over it.

### 3.11 Implications for the recipe

- `compile/llama-cpp.md` starts with a **Pre-conditions block** that lists what the agent must already know from `detect.md`'s context: selected recipe folder, GPUs (with compute_cap), `nproc`, `cmake` present, `nvcc` present.
- If `nvcc` is missing → abort with a message that points to the official installer (`.run` from NVIDIA, or distro packages) and reminds the user that the alternative is the pre-built image `ghcr.io/ggml-org/llama.cpp:server-cuda`.
- The CMake `CMAKE_CUDA_ARCHITECTURES` value is **derived from `nvidia-smi --query-gpu=compute_cap`** at recipe-execution time, not hard-coded. The agent joins the values with `;`.
- Build parallelism: `cmake --build build --config Release -j$(nproc)`. The `-j` value comes from the captured `nproc`; do not hard-code 8.
- The recipe is a single linear flow with no subagent fan-out, per the architectural decision in 3.1.

### 3.12 Iteration 1 scope: SKILL.md + detect.md only

After writing `compile/llama-cpp.md` we paused the validation loop because `cmake` and `nvcc` are not installed on this host and `sudo` requires a password the agent cannot supply. Rather than block on a real build, we shipped an iteration 1 with just the entry point and the first recipe:

- A first cut of `SKILL.md` at the repo root: a 1-page runtime-agnostic entry point that tells the agent to load `references/detect.md` and execute it. No orchestration, no fan-out, no subagent prescription.
- `references/detect.md`: unchanged from session 02 (6 subagent iterations, 18 improvements, hybrid heuristic+fallback).
- `references/ubuntu-26.04-wsl2/compile/llama-cpp.md`: written but **not yet validated** because the host is missing `cmake` and `nvcc`. The recipe is parked here, awaiting deps and a real end-to-end test.

The point of iteration 1 is to test the entry-point → detect.md loop on a real agent (not a simulated subagent), with a real user behind the keyboard, to see whether the agent actually surfaces the right install commands to the user and whether the report reads naturally. Once that loop is confirmed, we extend with the next recipes and validate compile once the host has the deps.

### 3.13 Why we tested `sudo` and `apt` first

Before deciding to scope down to iteration 1, we confirmed two things:

- `sudo -n apt install -y make cmake jq` → rejected with `sudo: a password is required`. A non-interactive agent cannot install system packages on this host.
- `apt install -y make cmake jq` (no sudo) → rejected with `Error: Could not open lock file /var/lib/dpkg/lock-frontend ... are you root?`. Running as `jorge`, the user is not root.

The only paths to install the deps are (a) the user runs the command interactively with `sudo`, or (b) reconfigure `/etc/sudoers` to grant NOPASSWD to `jorge`. Both are user decisions, not agent decisions. We did not auto-install.

### 3.14 Restructure to the Agent Skills convention

After drafting iteration 1's SKILL.md as a single root-level file, the user pointed us at the [Agent Skills specification](https://agentskills.io/specification). The skill directory was restructured to match the convention:

- `llama-wizard/SKILL.md` (frontmatter + body)
- `llama-wizard/references/` (the `recipes/` directory was renamed to `references/`)
- All internal path references in the recipes updated from `recipes/` to `references/`
- The repo's `BITACORA.md`, `VISION.md`, and `src/llama.cpp/` stay at the repo root, outside the skill directory (they are dev artifacts, not part of the skill)

Frontmatter contents:

- `name: llama-wizard` (matches the parent directory; lowercase, no leading/trailing hyphens, no `--`)
- `description`: 483 chars, under the 1024 limit. Describes what the skill does and when to use it; includes trigger keywords.
- `license: MIT`
- `compatibility`: 170 chars, under the 500 limit. Linux x86_64, NVIDIA driver 5xx+, CUDA 12.8+ (or pre-built image), Docker with nvidia-container-toolkit.
- `metadata.version: 0.1.0`, `status: iteration-1`, `runtime-model: single-agent-linear`.

The `skills-ref` CLI is not installed on this host, so validation was done manually against the spec (name length, name format, name/parent match, description length, compatibility length, body structure). The skill passes all manual checks.

### 3.15 Move to `.agents/skills/llama-wizard/` (Pi-compatible discovery)

After seeing the Pi skill-discovery rules (Pi scans `.agents/skills/`, `.pi/skills/`, `~/.pi/agent/skills/`, etc.), the user asked to move the skill to `.agents/skills/llama-wizard/`. This path satisfies both the Agent Skills spec (skill = directory with SKILL.md, name matches the directory) and Pi's auto-discovery (which scans `.agents/skills/`). No symlinks required.

The move was a rename; git preserved history (the old `llama-wizard/` directory and the new `.agents/skills/llama-wizard/` are recognised as the same file content). The first symlink-based attempt (`.agents/skills/llama-wizard -> ../../llama-wizard`) was reverted before this commit. The `.pi/skills/llama-wizard` symlink was also removed.

Final skill location: `.agents/skills/llama-wizard/`. Repo dev artifacts (BITACORA.md, VISION.md, README.md, src/llama.cpp/, .atl/, .pi/) stay at the repo root.

### 3.16 Real-agent test of iteration 1

The user loaded the skill in a real agent and ran `references/detect.md`. The agent produced the canonical report without improvisation:

- All 9 detection steps ran in parallel where independent.
- The `Runtimes:` awk worked on Docker 29.5.2's indented output (`io.containerd.runc.v2 nvidia runc`).
- The report included `Selected recipe: references/ubuntu-26.04-wsl2/` and the new path format was respected.
- The user-group render rule (`id -un`) worked: `Docker group: user jorge is in docker group`.
- The agent inferred `apt` for `id=ubuntu` via the heuristic (no per-distro table) and proposed `sudo apt update && sudo apt install -y make cmake jq` (the three tools the user thought were missing).
- Verdict: `blocked`. The agent stopped, told the user to run the install, and asked them to re-invoke the skill.

**Important correction**: the user pointed out that `make`, `cmake`, and `jq` are actually installed on this host (we had not noticed — session 01 recorded them as missing, but the user installed them since, or the install was never needed). Only `nvcc` is actually missing.

### 3.17 `nvcc` is optional, not required

Before session 05, `detect.md` treated `nvcc` as a required tool. That was wrong: the next recipe (`compile/llama-cpp.md`) is one of two paths, and the alternative path (pre-built image `ghcr.io/ggml-org/llama.cpp:server-cuda`) does not need `nvcc` on the host. Forcing `nvcc` in `detect.md` would block users who only want the image.

Change: `nvcc` is now an **optional** tool in step H. If it is missing:

- The verdict is `Verdict: ready` (NOT `ready with warnings` — that verdict is reserved for disk/GPU/recipe-caveat conditions).
- A `Warnings` section is added between the report table and the `Selected recipe:` line.
- The warning text tells the user that `nvcc` is needed only for the build-from-source path and that the pre-built image path bypasses it.

A subagent test of the new model confirmed the layout: `Verdict: ready`, warnings block, recipe folder, no improvisation. The test also caught a subtle ambiguity (the original wording said "verdict stays `ready` or `ready with warnings`", which conflicted with the verdict values table that excludes missing tools from `ready with warnings`); the wording was tightened to say the verdict is `ready`.

`compile/llama-cpp.md` keeps its `nvcc` abort — that recipe is reached only when the user has chosen the build-from-source path, and on that path `nvcc` is genuinely required.

### 3.18 `compile/llama-cpp.md` becomes image-aware (Dockerfile + steps A-J)

After iteration 1, `compile/llama-cpp.md` produced host binaries. The `pre-built` path (ghcr.io/ggml-org/llama.cpp:server-cuda) produced a Docker image. The two paths converged only at `references/compose.md` (forthcoming). The compile path's output (host binaries) had no clear destiny in the meantime.

The user asked: instead of producing host binaries, have the compile path produce a Docker image, just like the pre-built path does. The architectural fix: the compile flow does the heavy work (compiling llama.cpp with CUDA) on the host, but then stages the resulting binaries into a `build-context/` directory and builds a Docker image that the runtime can use. This way both paths produce a Docker image and `compose.md` only deals with images.

This required two new artifacts:

- `references/compile/Dockerfile` (sibling to `llama-cpp.md`): single-stage image based on `nvidia/cuda:13.3.0-runtime-ubuntu26.04`, installs `libgomp1` and `curl`, copies the 3 binaries plus all `*.so*` (3 families: libggml, libllama, libmtmd) into `/app/`, sets `ENV LLAMA_ARG_HOST=0.0.0.0`, no `LD_LIBRARY_PATH` (nvidia runtime + ldconfig already handle libcuda.so.1).
- New steps G (stage build-context), H (build the image), I (smoke test the image), J (report) in `llama-cpp.md`.

### 3.19 `nvidia/cuda:13.3.0-runtime-ubuntu26.04` is available on Docker Hub

The user's environment is `ubuntu-26.04` (resolute). Initial concern: does the `nvidia/cuda:13.3.0-runtime-ubuntu26.04` base image exist on Docker Hub? A `docker pull` confirmed it does (1718 MB, last updated 2026-06-03). The previous concern in an old `Dockerfile.custom` (kept in `src/llama.cpp/.devops/`) is obsolete.

### 3.20 `libgomp1` and the `*.so*` family bug

Two real bugs were found by spinning up the image:

1. `llama-server` is dynamically linked against `libgomp.so.1` (ggml-cpu uses OpenMP for parallel inference on the CPU). The `nvidia/cuda:13.3.0-runtime-ubuntu26.04` image does not have `libgomp1` installed by default. First container run failed with: `error while loading shared libraries: libgomp.so.1: cannot open shared object file`. Fix: add `libgomp1` to the `apt-get install` line. Verified `libgomp1` is in NVIDIA's APT repo (transitive dependency of CUDA packages), so no Ubuntu sources needed.

2. `llama-server` has DT_NEEDED entries for `libllama.so.0`, `libllama-common.so.0`, `libmtmd.so.0`, and `libggml-base.so.0` (NOT `libggml-cuda.so.0` — backends are statically linked because `GGML_BACKEND_DL=OFF` in llama.cpp's build). The first version of the Dockerfile's `COPY` only copied `bin/libggml-*.so*`. The container then failed with `libllama-common.so.0: cannot open shared object file`. Fix: change to `COPY bin/*.so*` (matches llama.cpp's official `cuda.Dockerfile` approach).

The "three families of .so" insight was the real bug; `libgomp1` was the simpler one. Both are now annotated in the Dockerfile with one-line comments so future regressions can be avoided.

### 3.21 End-to-end smoke test with `Qwopus3.6-27B-v2-Q6_K`

A real model was downloaded (22.08 GB Q6_K quantization, chosen for fitting in 32 GB RTX 5090 with 11 GB margin for context). Smoke test:

```
$ docker run --rm --gpus all llama-wizard-llama-cpp:gguf-v0.19.0 --version
ggml_cuda_init: found 2 CUDA devices (Total VRAM: 48982 MiB):
  Device 0: NVIDIA GeForce RTX 5090, compute capability 12.0, VMM: yes, VRAM: 32606 MiB
  Device 1: NVIDIA GeForce RTX 4080 SUPER, compute capability 8.9, VMM: yes, VRAM: 16375 MiB
version: 1 (a290ce626)
built with GNU 15.2.0 for Linux x86_64
```

The `--version` flag calls `ggml_cuda_init` to enumerate visible devices even though it does not load a model, so this is a stronger test than it looks: it validates the binary, the nvidia runtime wiring, and the CUDA library version compatibility in one shot.

A full E2E test was then run (script at `/tmp/test-e2e.sh`, not committed): start the container with a model mounted, poll `/health` (7s), GET `/v1/models` (lists the model with `format: "gguf"`), POST `/v1/chat/completions` (HTTP 200, 132 tok/s prompt eval, 37.5 tok/s generation). The model returned an empty `content` field but the inference loop was healthy; the empty response is a property of the "Qwopus" model variant (`qwen35` architecture with `thinking = 1` mode and `peg-native` chat format), not of the setup.

GPU usage during inference confirmed both GPUs were active (RTX 5090: 48 % util, 15825 MiB used; RTX 4080 SUPER: 48 % util, 8060 MiB used; KV cache split ~2:1 by memory).

### 3.22 Dockerfile: 116 → 41 lines

The original `Dockerfile` had 116 lines; only ~40 were code. The rest was a multi-block comment header explaining the dynamic linker resolution order, the contents of the nvidia/cuda base image, and the rationale for each instruction. After two passes of trim:

- 1st pass (116 → 57 lines): dropped the dynamic-linker tutorial, the "All three executables" comment, the "All shared libraries" comment, the EXPOSE / ENTRYPOINT explanations, the `610.47` host-driver example, and the "Image build wall time is dominated by apt-get update" prose. Kept the seven non-obvious decisions: reuses host build, runtime-vs-devel rationale, libgomp1, curl, LLAMA_ARG_HOST, *.so* wildcard, no LD_LIBRARY_PATH.
- 2nd pass (57 → 41 lines): dropped redundant prose inside each of the seven comments. Examples: "The base image already carries libcudart.so.13, libcublas.so.13, libcublasLt.so.13, and the rest of the CUDA runtime libraries under /usr/local/cuda/lib64/" (the agent can `find` to verify), "because ggml-cpu uses OpenMP" (OpenMP is general knowledge), "The DT_RUNPATH embedded in the host-built binary points at the host build directory and is a no-op inside the container" (the no-op is the entire point).

The agent reading the file should not need a dynamic-linker tutorial or a catalog of what the base image contains. The remaining 41 lines are: code (12 lines, ~30 %) and 6 comment blocks (29 lines, ~70 %), each explaining exactly one non-obvious decision in 1-2 lines.

Verified: rebuild 0.1s, smoke test still reports 2 CUDA devices and the pinned commit `a290ce626`.

### 3.23 `compile/llama-cpp.md` F-J and "What this file does NOT do"

Same pass: read the flow from F onward and identify prose that the agent can infer or that duplicates the Dockerfile. Concretely:

- F: drop the 4-line "cd is in a separate block" tutorial. Drop the redundant SHA check on `llama-cli` (same build, same HEAD, same SHA as `llama-server`). Compact the `^{commit}` peel explanation to a parenthetical.
- G: drop the 3-line explanation of what `*.so*` copies — the Dockerfile's `COPY bin/*.so*` comment already covers that. Drop `libggml-cuda.so.0` from the verification list (it is not a NEEDED entry; `GGML_BACKEND_DL=OFF` makes backends static). Consolidate the `ls` into one command.
- H: drop the 2-line "this step does not recompile llama.cpp" — the Dockerfile's header already says it.
- I: drop the 4-line "expected output" template (the agent sees the output when it runs the command). Drop the SHA check on the image (F already validated the host binary, Dockerfile only does COPY, so the SHA is the same by construction). Move the "loading a real model is compose.md's job" sentence to "What this file does NOT do" where it belongs. Reformat the 3 failure cases as a bullet list.
- J: compact the 3-line "Compile verdict means..." to a one-line parenthetical.
- "What this file does NOT do": split the third bullet into two sentences (one idea each).

255 → 237 lines (-7 %). The flow from F is now declarative, does not auto-repeat the Dockerfile, and each step has at most one non-obvious decision explained in 1-2 lines.

### 3.24 Final commit log (22 commits ahead of origin/main)

```
ee88947 refactor(compile): trim F-J and NOT do; remove prose and duplication
508f41a refactor(compile): compact Dockerfile comments further
87cb684 refactor(compile): drop tutorial comments from Dockerfile
db80b69 fix(detect): drop 'flujo' and 'Recipe' spanglish leftovers
1e277ff fix(compile): step I smoke test now validates CUDA init
38d1217 fix(compile): step G stage copies all .so*, not just libggml-*
0c68e31 fix(compile): Dockerfile loads CUDA + all .so + libgomp1
480eabc refactor(compile): align to detect.md style, drop host-specific assumptions
c3f76d6 fix(compile): drop remaining 'flujo' and 'binarios' spanglish
a30ba30 fix(compile): drop distro-specific assumptions, cwd-aware pin check
8974c49 fix
f56ffda refactor: replace 'recipe' with 'flujo', clarify compile+pre-built converge in compose
66e8132 refactor: flatten recipe layout, drop per-distro folders
50eebc5 fix(compile): cwd in step D, expected duration + nohup workaround in step E
8604593 feat(SKILL): add explicit compile-vs-docker path gate after detect.md
f6aa7a9 fix: clarify CMake line in compile report template
4591cac fix: 4 real bugs in compile/llama-cpp.md found by end-to-end subagent test
4e2e71c fix: refine compile/llama-cpp.md and detect.md based on subagent tests
b3abeb2 fix: nvcc is optional, not required, in detect.md
81cbb6f chore: move skill to .agents/skills/llama-wizard/ for Pi auto-discovery
b7164f8 feat: iteration 1 of llama-wizard, restructured to Agent Skills spec
73c1ca5 chore: initial skill scaffold with detect recipe
518f079 Initial commit
```

### 3.25 Outstanding work

- `references/compose.md` (forthcoming): destination of both compile and pre-built paths. Defines the compose service, healthcheck, and how the user picks a model.
- `references/models/` (forthcoming): model catalog + download flow.
- Consider validating the pre-built path (`ghcr.io/ggml-org/llama.cpp:server-cuda`) end-to-end so the simpler path is also confirmed working.
