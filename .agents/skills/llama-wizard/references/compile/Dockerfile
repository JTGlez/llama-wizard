# syntax=docker/dockerfile:1.7
#
# Custom llama.cpp image built from the host's local build artifacts.
#
# This Dockerfile does NOT clone + build llama.cpp from source; the host
# already did that in the previous steps of the compile flow, and the
# resulting binaries and shared libraries are passed in as the build
# context. This makes the image build fast (under a minute) and reuses
# the validated build.
#
# Base image rationale
# --------------------
# `nvidia/cuda:13.3.0-runtime-ubuntu26.04` is the runtime-only image
# from NVIDIA. It already carries the CUDA runtime libraries under
# `/usr/local/cuda/lib64/` (libcudart.so.13, libcublas.so.13,
# libcublasLt.so.13, libcufft.so.12, etc.). The base image also
# provides `/usr/local/cuda/compat/libcuda.so.<ver>` as a forward-
# compatibility shim for host drivers newer than the one the image was
# built against. The host's NVIDIA driver (e.g. 610.47 in this
# environment) provides libcuda.so.1 at container start via the nvidia
# container runtime when `--gpus all` is passed; this image does not
# embed it.
#
# Why we install libgomp1 here
# ----------------------------
# llama-server is dynamically linked against libgomp.so.1 (the OpenMP
# runtime that ggml-cpu uses for parallel inference on the CPU). The
# nvidia/cuda runtime image is built from the NVIDIA CUDA APT repo, but
# libgomp1 is available in that repo as a transitive dependency of the
# CUDA packages. We install it explicitly so the binary's NEEDED entry
# is satisfied at container start.
#
# Why we install curl here
# ------------------------
# The compose flow (forthcoming) defines a HEALTHCHECK for the
# llama-server container that hits `http://localhost:8080/health` with
# curl. curl is the standard healthcheck client; it must be inside the
# image, not the host.
#
# Why we don't set LD_LIBRARY_PATH
# --------------------------------
# The dynamic linker resolves shared libraries in this order:
#   1. LD_LIBRARY_PATH (only if the binary has DT_RUNPATH, not DT_RPATH)
#   2. DT_RUNPATH embedded in the binary
#   3. ld.so.cache (populated by ldconfig from the system image)
#   4. Default search path (/lib, /usr/lib)
# The host-built llama-server embeds DT_RUNPATH pointing at the host
# build directory (which does not exist inside the container), so that
# entry is a no-op. The nvidia container runtime mounts the host's
# libcuda.so.1 into /usr/lib/x86_64-linux-gnu/ (and runs ldconfig to
# register it) when `--gpus all` is passed, so the dynamic linker
# finds libcuda.so.1 via ld.so.cache. The CUDA runtime libraries and
# the ggml/llama/mtmd shared libraries are picked up the same way.
# Setting LD_LIBRARY_PATH here would override ld.so.cache and is not
# needed for the happy path; if a future host has a much-newer driver,
# the nvidia runtime falls back to /usr/local/cuda/compat/ on its own.

ARG CUDA_VERSION=13.3.0
ARG UBUNTU_VERSION=26.04

FROM nvidia/cuda:${CUDA_VERSION}-runtime-ubuntu${UBUNTU_VERSION}

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        libgomp1 \
 && rm -rf /var/lib/apt/lists/*

# llama-server defaults to host 127.0.0.1. When run inside a container,
# 0.0.0.0 is what the user (or compose port-forward) wants. The compose
# file can still override this with the `command:` directive.
ENV LLAMA_ARG_HOST=0.0.0.0

WORKDIR /app

# All three executables built from the llama.cpp source tree.
COPY bin/llama-server      /app/llama-server
COPY bin/llama-cli         /app/llama-cli
COPY bin/llama-completion  /app/llama-completion

# All shared libraries built from the llama.cpp source tree. There are
# three families (libggml-*, libllama-*, libmtmd-*), each provided as a
# SONAME major symlink plus the versioned file. The wildcard matches
# every variant of every family. Place them alongside the executables
# in /app/ so the dynamic linker finds them via ld.so.cache (ldconfig
# is run by the base image at the end of every apt install).
COPY bin/*.so* /app/

# llama-server listens on 8080 by default. The compose file publishes
# this port to the host.
EXPOSE 8080

# The default entrypoint is llama-server. The compose file can pass
# extra arguments (e.g. -m /models/...gguf) via `command:` in the
# service definition.
ENTRYPOINT ["/app/llama-server"]
