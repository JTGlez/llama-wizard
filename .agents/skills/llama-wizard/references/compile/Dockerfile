# syntax=docker/dockerfile:1.7
#
# Reuses the host's validated llama.cpp build; does NOT recompile in
# the image. The build context (set up in compile/llama-cpp.md step G)
# carries the executables and shared libraries produced by the host
# build. Image build wall time is dominated by the `apt-get update` for
# the base image, not by anything in this Dockerfile.

# Runtime image, not devel: the binary is pre-built, no nvcc needed in
# the container. The base image already carries libcudart.so.13,
# libcublas.so.13, libcublasLt.so.13, and the rest of the CUDA runtime
# libraries under /usr/local/cuda/lib64/. libcuda.so.1 is NOT in the
# image; the nvidia container runtime injects it from the host driver
# when --gpus all is passed.
ARG CUDA_VERSION=13.3.0
ARG UBUNTU_VERSION=26.04

FROM nvidia/cuda:${CUDA_VERSION}-runtime-ubuntu${UBUNTU_VERSION}

# libgomp1 is not in the nvidia/cuda runtime image by default; without
# this, llama-server fails with "libgomp.so.1: cannot open shared object
# file" because ggml-cpu uses OpenMP. curl is for the compose-driven
# /health healthcheck.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        libgomp1 \
 && rm -rf /var/lib/apt/lists/*

# llama-server defaults to host 127.0.0.1, which is not reachable from
# outside the container. The compose file can still override this with
# `command:`.
ENV LLAMA_ARG_HOST=0.0.0.0

WORKDIR /app

COPY bin/llama-server      /app/llama-server
COPY bin/llama-cli         /app/llama-cli
COPY bin/llama-completion  /app/llama-completion

# The host build produces three families of shared libraries (libggml-*,
# libllama-*, libmtmd-*), each as a SONAME major symlink plus a
# versioned file. All three families are runtime NEEDED entries of
# llama-server. The *.so* wildcard matches every variant of every
# family; do not narrow it to a single family.
COPY bin/*.so* /app/

# We deliberately do NOT set LD_LIBRARY_PATH. The nvidia container
# runtime mounts the host's libcuda.so.1 into /usr/lib/x86_64-linux-gnu/
# and runs ldconfig; the dynamic linker finds everything via
# ld.so.cache. The DT_RUNPATH embedded in the host-built binary points
# at the host build directory and is a no-op inside the container.

EXPOSE 8080

ENTRYPOINT ["/app/llama-server"]
