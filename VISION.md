# VISION — llama-wizard

> Living design document.
> `BITACORA.md` is the operational source of truth (what we actually ran).
> This file explains **why** the skill has the shape it has.

---

## 1. Purpose

`llama-wizard` is a single, runtime-agnostic entry point (Claude Code, Codex, Pi Agent) for a user with basic AI experience but no LLM background to spin up a **local LLM server** on Docker, assuming a Linux system.

What the skill **is**:
- A guided orchestrator that detects the environment, validates prerequisites, compiles `llama.cpp`, containerises it, downloads a curated model, and exposes an HTTP endpoint.
- A collection of `recipes/` per distribution, **manually validated** before being registered.

What the skill **is not**:
- A magic installer. The user sees and runs real commands; there is no "next-next-finish".
- A multi-backend swiss-army knife on day 1. We start with `llama.cpp` and grow with evidence.
- An infinite model catalog. We keep a small, curated set.

---

## 2. Why this design and not the original proposal literally

The initial proposal (tree with `models/`, `vendor/`, `recipes/`, `SKILL.md`, 8-step workflow) was a good sketch, but it had eight concrete problems. Listed so we do not forget why the final shape diverges.

### 2.1 "Validate distro and GPU count" must not ask the user what is detectable

Distro comes from `/etc/os-release`. GPU comes from `lspci` + `nvidia-smi` / `rocminfo`. **Detect first, ask after**. If detection fails, abort with a corrective action; never proceed blind.

### 2.2 "VENDOR" is a placeholder with no defined content

"Vendor" mixed `llama.cpp` (compiled), `ollama` (packaged binary), and `vllm` (Python + CUDA). They are not interchangeable. **Decision**: start with `llama.cpp` only (most portable, compiles CPU-only, scales to GPU, exposes OpenAI-compatible API). Rename the concept to `BACKEND` when a second one is added.

### 2.3 The model catalog is the Achilles' heel

"Ask the user which model to download" without context is a bomb. Models range from 4 GB to 200+ GB, have different licences, require specific VRAM and quantisation. **Decision**: closed catalog of 3-5 models with `name, repo, default quant, min VRAM, context, licence, use case`.

### 2.4 Happy-path only, no error handling or re-entry

What if a vendor is already compiled? What if the port is busy? What if the model does not fit in VRAM? **Decision**: the filesystem is the source of truth. Each recipe starts with a self-guard that checks its own precondition on disk (e.g. "compile only if `vendor/llama.cpp/build/bin/llama-server` does not exist"). No JSON state needed for day 1. Re-entry on day 2+ is solved by the agent reading what is on disk, not by parsing a state file.

### 2.5 The "agnostic" SKILL.md is not trivial

Claude Code, Codex, and Pi do not share the same frontmatter. Assuming compatibility without verifying is wishful thinking. **Decision**: the SKILL.md is validated in the three runtimes before declaring it agnostic. If they differ, two frontmatter blocks live in the same file, separated by a comment line.

### 2.6 Detection ≠ prerequisite validation

The original step 2 mixed "what distro are you on" (detection = read) with "is everything installed" (validation = probe existence, minimum version, permission). **Decision**: separate. `detect.md` reads; `requirements.md` probes and aborts if a critical item is missing.

### 2.7 The step 8 smoke test was underspecified

The "base endpoint" depends on the backend:
- `llama.cpp` (`llama-server`) → `POST /v1/chat/completions` (OpenAI-compatible)
- Ollama → `POST /api/chat` or `POST /api/generate`
- vLLM → `POST /v1/chat/completions`

**Decision**: the smoke test payload is bound to the chosen backend, never generic. It lives inside `recipes/<DISTRO>/compose.md`, in the final "validate" section.

### 2.8 System prerequisites that were missing

`docker` group, `nvidia-container-toolkit` (NVIDIA), ROCm (AMD), disk space, `git`, `make`, `cmake`, `gcc`, `curl`, `jq`. The Ubuntu recipe differs from Fedora on this. **Decision**: `requirements.md` per distro enumerates the exact packages and validation commands.

---

## 3. The four principles

Ordered by priority, not by implementation order.

### 3.1 Detect first, ask after
`detect.md` runs always. Only ask the user what could not be inferred (preferred port, model from catalog, etc.). If detection fails, abort with an actionable message, do not proceed.

### 3.2 Closed catalog at start, grow with evidence
1 backend (`llama.cpp`) and 3-5 models on day 1. Do not promise 5 backends on day 1. Each new backend or model is added **after** we have a reproducible run recorded in `BITACORA.md`.

### 3.3 Filesystem is the source of truth
No `state.json` for day 1. Each recipe has a self-guard at the top that checks its own precondition on disk. Re-entry after a closed session works because the agent reads the filesystem, not a state file. If a state file becomes valuable later (multi-backend, multi-model, very long flows), we add it then.

### 3.4 BITACORA is the contract for writing recipes
Each `BITACORA.md` entry is a timestamped record of: command, output, result, decision. **Recipes are written after we run the flow manually and record it**. Without a bitacora, recipes are aspirational.

---

## 4. Repository structure

```
llama-wizard/
├── SKILL.md                                  # runtime-agnostic entry point
├── VISION.md                                 # this file
├── BITACORA.md                               # operational source of truth
├── README.md                                 # what it is, what it is not, current state
├── recipes/
│   └── <DISTRO>/                             # e.g. ubuntu-26.04-wsl2/, ubuntu-24.04/, fedora-41/
│       ├── detect.md                         # declarative environment detection steps
│       ├── requirements.md                   # packages and validation commands
│       ├── compile/
│       │   └── llama-cpp.md                  # clone + compile steps, manually tested
│       ├── docker/
│       │   └── llama-cpp.md                  # build the image with llama-server baked in
│       ├── models/
│       │   └── CATALOG.md                    # 3-5 curated models with full metadata
│       ├── download.md                       # download a model from the catalog
│       └── compose.md                        # docker compose up + smoke test payload
├── models/                                   # downloaded model weights (gitignored)
└── vendor/                                   # cloned source trees (gitignored)
```

No `lib/`. No scripts. No `state.json`. Everything is declarative markdown that any compatible agent can read and execute.

### 4.1 Recipe naming

Distro folders are identified by `ID` and `VERSION_ID` combined, not just by `ID`:
- `ubuntu-24.04` (noble)
- `ubuntu-26.04-wsl2` (resolute + WSL2 flag; recipe specific to the kernel and GPU passthrough)
- `fedora-41`

This is because **Ubuntu 24.04 native** and **Ubuntu 24.04 on WSL2** have operational differences (GPU passthrough, driver paths, Docker Desktop vs Docker Engine, `systemd`). A single `ubuntu/` does not cover both.

### 4.2 Runtime folders

`models/` and `vendor/` live at the repo root, not inside `recipes/`. The catalog is metadata in `recipes/<DISTRO>/models/CATALOG.md`; the binaries live in `models/` and `vendor/` respectively.

---

## 5. Revised flow

```
1. SKILL.md → load recipes/<DISTRO>/detect.md (chosen by ID + VERSION_ID)
2. detect.md → run declarative detection steps; record results in BITACORA.md
3. If detection fails (not Linux, distro without recipe) → abort with actionable message
4. SKILL.md → load recipes/<DISTRO>/requirements.md
5. requirements.md → validate prerequisites (docker running, nvidia-container-toolkit, disk space, packages)
6. If a critical item is missing → abort and say "install X with Y"
7. SKILL.md → ask the user which backend to compile (only those with a recipe; today: llama.cpp only)
8. recipes/<DISTRO>/compile/llama-cpp.md → self-guard; if not compiled, clone and compile
9. recipes/<DISTRO>/docker/llama-cpp.md → docker build (with the binary from step 8)
10. recipes/<DISTRO>/models/CATALOG.md → present catalog; ask the user
11. recipes/<DISTRO>/download.md → download chosen model into models/
12. recipes/<DISTRO>/compose.md → docker compose up -d
13. compose.md (validate section) → POST a smoke test payload to the OpenAI-compatible endpoint
14. Record a summary entry in BITACORA.md
```

Each step of the real manual run goes into `BITACORA.md` with timestamp, command, summarised output, result, and notes. **That bitacora is the input for writing the final recipes** — not the other way around.

---

## 6. Decisions taken in session (2026-06-07)

- **Initial backend**: `llama.cpp` only.
- **Detect first, ask after**. Never ask the user for distro or GPU.
- **Closed catalog** of models at start (defined later in `recipes/<DISTRO>/models/CATALOG.md` after the detection phase).
- **No `state.json`**. Filesystem is the source of truth; each recipe self-guards.
- **BITACORA first**. Before writing the SKILL.md, we run the flow manually and dump to `BITACORA.md`.
- **No scripts, no `lib/`**. Everything is declarative markdown.
- **Smoke test payload lives inside `compose.md`**, in its own "validate" section.

### 6.1 Iteration environment finding

The author's iteration environment is **Ubuntu 26.04 LTS (Resolute Raccoon) on WSL2**, not Ubuntu 24.04 native. Implications:

- Ubuntu 26.04 is **very recent** (pre-release at the time of this writing). Some `apt` packages may not be available or may be named differently than on 24.04. We must `apt-cache search` by name before fixing the install command.
- WSL2 specifics: GPU passthrough requires `nvidia-container-toolkit` with WSL2 CUDA support enabled, and the device path (`/dev/dxg` on WSL2) differs from native Linux. The `ubuntu-26.04-wsl2/` recipe must reflect this.
- The host kernel is 6.6.114.1-microsoft-standard-WSL2. If any step assumes active `systemd`, it will fail in WSL2 (unless WSL2 is explicitly configured with systemd).
- **Consequence on promised portability**: the `ubuntu-26.04-wsl2/` recipe is not directly portable to `ubuntu-24.04/` native. Either we validate portability explicitly, or we build two recipes. Decision deferred until the Ubuntu 26.04 WSL2 recipe is working end-to-end.

### 6.2 Next concrete step

Run the detection commands manually in the author's machine and record the output in `BITACORA.md` as the first entry. That defines distro, kernel, WSL flag, GPU, drivers, Docker, disk space, and tools, and gives us real input for `detect.md` and `requirements.md`.

---

## 7. Open questions (to resolve in later sessions)

- Which 3-5 models go in the catalog? (Depends on the GPU the author has and the use case. Resolved after detection.)
- Does the Docker image bake the compiled binary (multi-stage build) or mount it as a volume? Multi-stage is cleaner and more portable; volume is faster to iterate. Decision at the Docker step.
- Do we want a `Makefile` or `justfile` at the root for `detect`, `requirements`, `compile`, `docker`, `download`, `compose` targets? (Lean: yes, reduces friction.) But this conflicts with the "no scripts" rule. Defer.
- Should the skill offer "uninstall" / "reset"? Yes, eventually. Not on day 1.
- Recipe versioning? (Each time a recipe's content changes, bump version and note in `BITACORA.md`.)
- How do we validate that `SKILL.md` is truly runtime-agnostic? (Test in Claude Code, Codex, and Pi on the same machine.)
