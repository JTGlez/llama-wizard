# syntax=docker/dockerfile:1.7
#
# Custom llama.cpp image built from the host's local build artifacts.
#
# This Dockerfile does NOT clone + build llama.cpp from source; the host
# already did that in the previous steps of the compile flujo, and the
# resulting binaries are passed in as the build context. This makes the
# image build fast (under a minute) and reuses the validated build.
#
# Stage 1 is the runtime image. It starts from NVIDIA's CUDA runtime
# image (which carries libcudart, libcublas, libcublasLt, etc.),
# installs ca-certificates, and copies the binaries + .so files from
# the build context into /usr/local. ldconfig is run so the dynamic
# linker finds the .so files without an LD_LIBRARY_PATH workaround.
#
# The host's NVIDIA driver (e.g. 610.47 in this environment) provides
# libcuda.so.1 at runtime via the nvidia container runtime; this image
# does not embed it.

ARG CUDA_VERSION=13.3.0
ARG UBUNTU_VERSION=26.04

FROM nvidia/cuda:${CUDA_VERSION}-runtime-ubuntu${UBUNTU_VERSION}

ARG LLAMA_CPP_REF=gguf-v0.19.0
ENV LLAMA_CPP_REF=${LLAMA_CPP_REF}

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
        ca-certificates \
 && rm -rf /var/lib/apt/lists/*

COPY bin/llama-server      /usr/local/bin/llama-server
COPY bin/llama-cli         /usr/local/bin/llama-cli
COPY bin/llama-completion  /usr/local/bin/llama-completion
COPY bin/libggml-*.so*     /usr/local/lib/

RUN ldconfig

# llama-server listens on 8080 by default. The compose file (forthcoming)
# publishes this port to the host.
EXPOSE 8080

# The default entrypoint is llama-server. The compose file can pass
# extra arguments (e.g. -m /models/...gguf --host 0.0.0.0) via
# `command:` in the service definition.
ENTRYPOINT ["/usr/local/bin/llama-server"]
