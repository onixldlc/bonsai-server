# Ternary Bonsai 2 27B server — PrismML llama.cpp fork (PQ2_0 / PTQ1_0 ggml types 142/143).
# Stock llama.cpp cannot load these files, so we ship the fork's prebuilt CUDA binaries.
# Base and release asset switch together: CUDA for the GPU build, plain ubuntu for the
# CPU-only build (see compose.cpu.yaml). Defaults are the GPU build.
ARG BASE_IMAGE=docker.io/nvidia/cuda:12.8.1-runtime-ubuntu24.04
FROM ${BASE_IMAGE}

# Release of https://github.com/PrismML-Eng/llama.cpp and which asset of it to install.
ARG PRISM_TAG=prism-b10754-2459f68
ARG PRISM_FLAVOR=linux-cuda-12.8-x64

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update

RUN apt-get install -y --no-install-recommends ca-certificates curl tar libgomp1 libcurl4 jq

RUN rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN curl -fsSL -o /tmp/llama.tar.gz "https://github.com/PrismML-Eng/llama.cpp/releases/download/${PRISM_TAG}/llama-${PRISM_TAG}-bin-${PRISM_FLAVOR}.tar.gz"

RUN mkdir -p /app/bin

RUN tar -xzf /tmp/llama.tar.gz -C /app/bin --strip-components=1

RUN rm -f /tmp/llama.tar.gz

RUN /app/bin/llama-server --version || true

COPY docker/entrypoint.sh /usr/local/bin/entrypoint.sh
COPY docker/verify.sh /usr/local/bin/verify.sh

# MCP stdio server for web_search / web_fetch. llama-server spawns it as a
# child process, so it ships in the image rather than as a separate service.
COPY docker/mcp-web.sh /usr/local/bin/mcp-web
COPY docker/mcp.json.template /etc/bonsai2/mcp.json.template

RUN chmod +x /usr/local/bin/entrypoint.sh /usr/local/bin/verify.sh /usr/local/bin/mcp-web

# The fork's tarball carries its own shared objects next to the binaries.
ENV LD_LIBRARY_PATH=/app/bin:/usr/local/cuda/lib64
ENV PATH=/app/bin:$PATH
# Runtime knobs, all overridable from compose / --env.
# Weights are pulled with plain resumable curl from the HF CDN.
ENV MODEL_BASE_URL=https://huggingface.co
ENV MODEL_REPO=prism-ml/Ternary-Bonsai-2-27B-gguf
ENV MODEL_FILE=Ternary-Bonsai-2-27B-PQ2_0.gguf
ENV MMPROJ_FILE=Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf
ENV ENABLE_VISION=1
ENV MODEL_DIR=/models
ENV CTX_SIZE=32768
ENV N_GPU_LAYERS=99
# 0 on the CPU build; empty means "let llama.cpp pick" (cores - 1 is usually right).
ENV THREADS=
ENV THREADS_BATCH=
ENV FLASH_ATTN=on
ENV HOST=0.0.0.0
ENV PORT=8080
ENV DOWNLOAD_ATTEMPTS=20
ENV EXTRA_ARGS=
# Web tools, off by default. 1 renders the MCP config and hands llama-server
# --mcp-servers-config, exposing web_search / web_fetch to the model.
ENV MCP_ENABLE=0
# Empty = DuckDuckGo's lite endpoint, no key. Point at your own SearXNG to
# keep queries on your network (its settings.yml needs `json` in search.formats).
ENV SEARXNG_URL=
ENV SEARCH_RESULTS=8
ENV FETCH_MAX_CHARS=20000
ENV HTTP_TIMEOUT=20

EXPOSE 8080

VOLUME ["/models"]

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
