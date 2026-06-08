# syntax=docker/dockerfile:1.7
#
# Reuses the host's validated build; does NOT recompile. Build context
# is staged by compile/llama-cpp.md step G.

# Runtime, not devel (binary is pre-built, no nvcc needed). libcuda.so.1
# is injected by the nvidia runtime via --gpus all.
ARG CUDA_VERSION=13.3.0
ARG UBUNTU_VERSION=26.04

FROM nvidia/cuda:${CUDA_VERSION}-runtime-ubuntu${UBUNTU_VERSION}

# libgomp1: not in the runtime image by default; ggml-cpu uses OpenMP.
# curl: for compose's /health healthcheck.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        libgomp1 \
 && rm -rf /var/lib/apt/lists/*

# Default 127.0.0.1 is not reachable from outside the container; compose
# can override.
ENV LLAMA_ARG_HOST=0.0.0.0

WORKDIR /app

COPY bin/llama-server      /app/llama-server
COPY bin/llama-cli         /app/llama-cli
COPY bin/llama-completion  /app/llama-completion

# Three families (libggml-*, libllama-*, libmtmd-*), all NEEDED by
# llama-server. The wildcard matches all; do not narrow.
COPY bin/*.so* /app/

# LD_LIBRARY_PATH is deliberately not set: the nvidia runtime mounts
# libcuda.so.1 and runs ldconfig.

EXPOSE 8080

ENTRYPOINT ["/app/llama-server"]
