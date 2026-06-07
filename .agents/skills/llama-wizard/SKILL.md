---
name: llama-wizard
description: A runtime-agnostic skill for setting up a local LLM server with llama.cpp on the user's host. Use when the user wants a self-hosted LLM inference server with GPU acceleration on Linux. Triggers on requests like "set up a local LLM", "run llama.cpp", "start a llama server", "I want a local AI server".
license: MIT
compatibility: Linux x86_64 host, NVIDIA GPU with driver 5xx+ and CUDA Toolkit 12.8+ (or use the pre-built image path), Docker with nvidia-container-toolkit installed, 20+ GiB free disk
metadata:
  version: 0.1.0
  status: iteration-1
  runtime-model: single-agent-linear
---

# llama-wizard

A runtime-agnostic skill for setting up a local LLM server with `llama.cpp` on the user's host. The skill detects the host's distro and environment, picks the right recipe, and walks the user through every step until a working server is up.

## How this skill works

This skill is designed to be executed by a **single LLM agent** with a `bash` tool. The agent reads each file in turn, runs the bash commands inside, and accumulates context across files.

The skill directory follows the [Agent Skills](https://agentskills.io/specification) convention:

- `SKILL.md` (this file) — entry point and metadata
- `references/` — additional markdown files the agent loads on demand, one per step of the flow

## Flow

1. **Detect environment.** Load and execute `references/detect.md`. It runs a series of bash commands to identify the host's distro, kernel, CPU, RAM, GPUs, Docker, and tools. It produces a verdict and an "Environment report".
2. **If the verdict is `blocked`** (most commonly: missing build tools or missing Docker GPU runtime), the recipe also produces install commands for the user to run manually. The skill ends here. The user runs the commands, re-invokes the skill, and the cycle repeats.
3. **If the verdict is `ready` or `ready with warnings`**, the agent stops and tells the user which recipe folder was selected. The next recipes (compile, docker, models, compose) are not yet implemented in this iteration.

## What this skill does not do

- It does not install system packages on its own. When prerequisites are missing, it tells the user the exact command to run and asks them to re-invoke the skill.
- It does not orchestrate subagents or hand off context between separate agent invocations. The model is "one agent, one linear flow".
- It does not assume a specific runtime. It is designed to work in any LLM agent that has a `bash` tool and can read files.
- It does not write any state to disk. Each recipe is self-guarding and idempotent: re-running it must be safe.
