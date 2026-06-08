---
name: llama-wizard
description: A runtime-agnostic skill for setting up a local LLM server with llama.cpp on the user's host. Use when the user wants a self-hosted LLM inference server with GPU acceleration on Linux. Triggers on requests like "set up a local LLM", "run llama.cpp", "start a llama server", "I want a local AI server".
license: MIT
compatibility: Linux x86_64 host, NVIDIA GPU with driver 5xx+ and CUDA Toolkit 12.8+ (or use the pre-built image path), Docker with nvidia-container-toolkit installed, 20+ GiB free disk
metadata:
  version: 0.3.0
  status: iteration-3
  runtime-model: single-agent-linear
  validated-flows: detect, compile/llama-cpp
  forthcoming-flows: compose, models/download
  flow-layout: flat (flows are distro-agnostic; no per-distro folders)
---

# llama-wizard

A runtime-agnostic skill for setting up a local LLM server with `llama.cpp` on the user's host. The skill detects the host's environment, presents a path gate, and walks the user through one of two flows that both converge on a running server.

## How this skill works

This skill is designed to be executed by a **single LLM agent** with a `bash` tool. The agent reads each file in turn, runs the bash commands inside, and accumulates context across files.

The skill directory follows the [Agent Skills](https://agentskills.io/specification) convention:

- `SKILL.md` (this file) — entry point and metadata
- `references/` — additional markdown files the agent loads on demand, one per step of the flow

## Flow

1. **Detect environment.** Load and execute `references/detect.md`. It runs a series of bash commands to identify the host's distro, kernel, CPU, RAM, GPUs, Docker, and tools. It produces a verdict and an "Environment report".
2. **If the verdict is `blocked`** (most commonly: missing build tools or missing Docker GPU runtime), the flow produces install commands for the user to run manually. The skill ends here. The user runs the commands, re-invokes the skill, and the cycle repeats.
3. **If the verdict is `ready` or `ready with warnings`**, present a **path gate** to the user. Both paths converge on `references/compose.md` (forthcoming), which is where the server actually runs. The gate is a choice of *how* the image the server runs on was built:

   | Path | What runs the server | Where the image comes from | Prerequisites |
   | --- | --- | --- | --- |
   | **compile** (Recommended when you want to tune build flags) | `references/compose.md` (forthcoming), using a custom image | `references/compile/llama-cpp.md` builds a custom Docker image with llama.cpp compiled from source at the pinned tag, with the host's CUDA archs. | `cmake` and `nvcc` (CUDA Toolkit) on `PATH`. The flow re-validates both and aborts with a distro-neutral link if either is missing. |
   | **pre-built** (Recommended when you just want a server) | `references/compose.md` (forthcoming), using the official image | The compose file pulls `ghcr.io/ggml-org/llama.cpp:server-cuda` directly. No host-side compile. | Docker with nvidia runtime (already validated in step 1). |

   The agent must present this gate to the user and wait for an explicit choice. Do not auto-pick. Do not silently switch paths. The user picked the path at the entry point; this gate is the same decision re-stated now that the verdict is known.

   Flows are distro-agnostic. The detection result (distro, kernel, WSL flag) is reported in step 1 but does not select a flow folder.

4. **Execute the chosen path.** The `compile` path goes to `references/compile/llama-cpp.md` and produces a custom image. The `pre-built` path skips straight to `references/compose.md`. Both end at compose.

5. **(forthcoming) Download a model.** Load and execute `references/models/download.md` to fetch a GGUF into the local `models/` directory. As this step is still not implemented, end the skill flow here for now.
6. **(forthcoming) Run the server.** Load and execute `references/compose.md` to start the server with a smoke test. This is the terminal step of the flow; the server stays up until the user stops it.

## What this skill does not do

- It does not install system packages on its own. When prerequisites are missing, it tells the user the exact command to run and asks them to re-invoke the skill.
- It does not orchestrate subagents or hand off context between separate agent invocations. The model is "one agent, one linear flow".
- It does not assume a specific runtime. It is designed to work in any LLM agent that has a `bash` tool and can read files.
- It does not write any state to disk. Each flow is self-guarding and idempotent: re-running it must be safe.
