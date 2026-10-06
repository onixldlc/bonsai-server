# Ternary Bonsai 2 27B — containerized llama-server

[![image](https://github.com/onixldlc/bonsai-server/actions/workflows/docker.yml/badge.svg)](https://github.com/onixldlc/bonsai-server/actions/workflows/docker.yml)

PrismML's Bonsai 2 27B ships as ternary GGUF (`PQ2_0` = ggml type 142, `PTQ1_0` = type 143).
**Stock llama.cpp, ollama, LM Studio and vLLM cannot load it** — the types are unmerged and
upstream has no Hadamard activation runtime, so it either refuses the file or emits garbage.
This image uses the prebuilt CUDA binaries from
[PrismML-Eng/llama.cpp](https://github.com/PrismML-Eng/llama.cpp) (branch `prism`).

| Piece | Value |
|---|---|
| Base model | Qwen3.8 27B, compressed to 1.76 bpw, Apache 2.0 |
| Weights | [`prism-ml/Ternary-Bonsai-2-27B-gguf`](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) |
| Fork release | [`prism-b10754-2459f68`](https://github.com/PrismML-Eng/llama.cpp/releases/tag/prism-b10754-2459f68), asset `bin-linux-cuda-12.8-x64` |
| Image | [`ghcr.io/onixldlc/bonsai-server`](https://github.com/onixldlc/bonsai-server/pkgs/container/bonsai-server) |
| API | OpenAI-compatible on `:8080` (`/v1/chat/completions`), web UI at `/` |

Links: [weights on HuggingFace](https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf) ·
[PrismML llama.cpp fork](https://github.com/PrismML-Eng/llama.cpp) ·
[llama-server docs](https://github.com/ggml-org/llama.cpp/tree/master/tools/server) ·
[GHCR package](https://github.com/onixldlc/bonsai-server/pkgs/container/bonsai-server) ·
[build workflow](.github/workflows/docker.yml)

## Layout

```
docker/        Dockerfile + entrypoint.sh + verify.sh — shared, identical for both stacks
gpu/           compose.yaml + .env  — CUDA build, RTX 3060
cpu/           compose.yaml + .env  — CPU-only build, for a VPS
VERSION        the release number; changing it is what triggers a publish
.github/       workflow that builds both variants and pushes them to GHCR
```

The Dockerfile takes `BASE_IMAGE` and `PRISM_FLAVOR` build args, so the same file produces
the CUDA image (`nvidia/cuda` base, 207 MB release asset) and the CPU image
(`ubuntu:24.04` base, 17.8 MB asset). Each stack's compose sets its own args; everything
that actually differs lives in that stack's `.env`.

Run a stack from inside its folder:

```bash
cd gpu && podman compose up -d --build     # 3060
cd cpu && podman compose up -d --build     # VPS
```

Both declare a volume named `bonsai2-models`, but compose prefixes it with the folder name,
so on one host they are `gpu_bonsai2-models` and `cpu_bonsai2-models` — separate 7.8 GB copies.
That only matters if you run both on the same machine.

## Prebuilt images

Skip the build and pull from GHCR. One image name, the variant is the tag:

| Tag | What it is |
|---|---|
| `ghcr.io/onixldlc/bonsai-server:gpu` | newest CUDA build — rolling |
| `ghcr.io/onixldlc/bonsai-server:cpu` | newest CPU-only build — rolling |
| `ghcr.io/onixldlc/bonsai-server:v0.1.0-gpu` | that release's CUDA build — pinned |
| `ghcr.io/onixldlc/bonsai-server:v0.1.0-cpu` | that release's CPU-only build — pinned |

```bash
podman run -d --name bonsai2 \
  --device nvidia.com/gpu=0 --security-opt label=disable \
  -v bonsai2-models:/models -p 8080:8080 \
  -e MODEL_FILE=Ternary-Bonsai-2-27B-PQ2_0.gguf -e CTX_SIZE=32768 \
  ghcr.io/onixldlc/bonsai-server:gpu
```

The compose stacks build locally on purpose (`image: localhost/bonsai2-server:...`). Point
`image:` at a GHCR tag and drop the `build:` block to run the published one instead.

## CI

[`.github/workflows/docker.yml`](.github/workflows/docker.yml) builds both variants in a
matrix off the one Dockerfile and pushes them to GHCR.

**The `VERSION` file is the trigger.** The workflow runs only on a push to `main` that
changes that file — any other commit costs zero Actions minutes. So a release is one edit,
made from anywhere: the GitHub web editor, github.dev, a phone. No git tag, no local clone,
no credentialed machine.

```
VERSION: 0.1.0 -> 0.2.0        commit to main
  => ghcr.io/onixldlc/bonsai-server:v0.2.0-gpu   (pinned)
     ghcr.io/onixldlc/bonsai-server:v0.2.0-cpu   (pinned)
     ghcr.io/onixldlc/bonsai-server:gpu          (moved to this build)
     ghcr.io/onixldlc/bonsai-server:cpu          (moved to this build)
```

The file holds the bare number (`0.2.0`); the workflow adds the `v`. Pushing the same
VERSION twice republishes over those tags, so bump it to keep a pinned build.

No registry secrets to set up — it logs in with the built-in `GITHUB_TOKEN`. Actions tab →
*image* → **Run workflow** forces a run without touching the file, with an optional
`version` override.

## Pick the quant

| VRAM | `MODEL_FILE` | Size | Usable context |
|---|---|---|---|
| 8 GB | `Ternary-Bonsai-2-27B-PTQ1_0.gguf` | 5.95 GB | ~16K |
| 12 GB | `Ternary-Bonsai-2-27B-PQ2_0.gguf` | 7.21 GB | ~56K |
| 16 GB+ | `Ternary-Bonsai-2-27B-PQ2_0.gguf` | 7.21 GB | 118K–262K |

PQ2_0 also wins prompt processing everywhere; PTQ1_0 wins decode on Ada/L4.

## Run

```bash
cp .env.example .env     # pick MODEL_FILE + CTX_SIZE for your card
podman compose up -d --build
podman logs -f bonsai2   # first start downloads 7.84 GB into the bonsai2-models volume
curl localhost:8080/v1/models
```

Plain podman, same shape as the invokeai run:

```bash
podman build -t bonsai2-server -f docker/Dockerfile .
podman run --rm --name bonsai2 \
  --device nvidia.com/gpu=0 --security-opt label=disable \
  -v bonsai2-models:/models -p 8080:8080 \
  -e MODEL_FILE=Ternary-Bonsai-2-27B-PQ2_0.gguf -e CTX_SIZE=32768 \
  bonsai2-server
```

`nvidia.com/gpu=0` is the RTX 3060 and nothing else — `/etc/cdi/nvidia.yaml` lists only that card,
so the AMD RX 9060 XT is never handed to the container. `nvidia-ctk cdi list` prints the valid names.
Your older `--runtime=nvidia --gpus '"device=0"'` form still works on the same image.

## CPU-only stack

For a VPS with no GPU. Same weights, same server, `-ngl 0`.

```bash
cd cpu
podman compose up -d --build
```

What is tuned differently in `cpu/.env`:

| Knob | Value | Why |
|---|---|---|
| `N_GPU_LAYERS` | `0` | everything on host cores |
| `CPU_LIMIT` | `3.0` | 50% of a 6 vCPU box |
| `THREADS` / `THREADS_BATCH` | `3` | matched to the quota |
| `ENABLE_VISION` | `0` | the vision tower is slow without a GPU |
| `CTX_SIZE` | `65536` | RAM is cheaper than VRAM |
| KV cache | `q8_0` | halves cache RAM for negligible quality cost |

### Capping CPU usage

`cpus:` is a CFS quota, not a core count. The kernel gives the container that much CPU *time*
per scheduling period and freezes it for the rest, so `CPU_LIMIT=3.0` on a 6 vCPU box is a hard
50% ceiling on total usage — it can never pin the whole machine, no matter how many threads
llama.cpp spawns. Scale it to the box: 4 vCPU → `2.0`, 8 vCPU → `4.0`.

Keep `THREADS` equal to `CPU_LIMIT`. Running 6 threads under a 3.0 quota is not faster than
running 3 — the same total CPU time gets split across twice as many threads, each of which is
frozen half the time, which wastes cache locality and adds latency.

`cpu_shares: 512` is a different dial: it only decides who wins when the host is contended,
leaving the hard cap untouched. `mem_limit` bounds RAM. Swap in `cpuset: "0-2"` for hard core
pinning if you want the inference confined to specific cores rather than given a time share.

Sizing: 7.21 GB weights + roughly 1.8 GB of q8_0 KV at 65536 + overhead — a 16 GB VPS works,
24 GB is comfortable. Decode speed on 6 vCPU will be a few tokens/second rather than the 21 t/s
the 3060 gives; it is bound by memory bandwidth, not clock speed, so a VPS with faster RAM beats
one with more cores. Measure yours before committing:

```bash
podman exec bonsai2-cpu llama-bench -m /models/Ternary-Bonsai-2-27B-PQ2_0.gguf -ngl 0 -t 5
```

## This machine (hybrid AMD + NVIDIA)

GPU 0 is an AMD RX 9060 XT driving the display; GPU 1 is the RTX 3060 (`0000:0d:00.0`,
`card1` / `renderD129`). The NVIDIA container toolkit only ever enumerates NVIDIA cards, so
inside the container the 3060 is device 0 and the AMD card is invisible — nothing to exclude.
Confirmed 12288 MiB and driver 615.71.09, so `.env` runs PQ2_0 with the vision tower.

With a hybrid-graphics manager in play, keep the 3060 out of runtime D3 power-off while the
container holds it — a suspended card shows up as a CUDA init failure at startup, not as a clean error.

## Getting the weights in

The entrypoint pulls each file with plain resumable `curl` straight from the HF CDN —
no `hf` CLI, no Python, no Xet. A stalled transfer resumes from the byte it reached
(`curl -C -` against a `.part` file) and retries up to `DOWNLOAD_ATTEMPTS` times; a transfer
that drops under 1 KB/s for 60 s is cut and resumed rather than left hanging.

**Or download them yourself** and drop them in — the entrypoint checks for the `GGUF` magic
bytes and skips anything already present:

```bash
curl -L -C - -O https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main/Ternary-Bonsai-2-27B-PQ2_0.gguf
curl -L -C - -O https://huggingface.co/prism-ml/Ternary-Bonsai-2-27B-gguf/resolve/main/Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf

podman run --rm -v bonsai2-models:/models -v "$PWD":/host:z \
  docker.io/library/busybox sh -c 'cp /host/*.gguf /models/'
```

Resume a half-finished host download by re-running the same `curl -C -` line.

## Weights volume

Weights live in the named volume `bonsai2-models`, so `compose down` / `up` never re-downloads.
It holds the `.gguf` files and nothing else: every HuggingFace cache path (`HF_HOME`,
`HF_HUB_CACHE`, `XDG_CACHE_HOME`) points at `/var/cache` inside the container, and the entrypoint
deletes the `.cache/huggingface` resume metadata that `hf download` leaves behind in the target dir.

```bash
podman volume inspect bonsai2-models --format '{{.Mountpoint}}'
podman run --rm -v bonsai2-models:/m docker.io/library/busybox ls -lA /m
podman volume rm bonsai2-models    # only if you want the 7.84 GB back
```

## Verify

```bash
podman exec -it bonsai2 verify.sh
```

Checks GPU passthrough, that the binary really is the fork (ternary types present),
weights on disk, and `/health`.

## Sampling

Thinking mode `--temp 1.0 --top-p 0.95 --top-k 20 --min-p 0.05`;
instruct mode `--temp 0.7 --top-p 0.80 --top-k 20`. Set via `EXTRA_ARGS`.
Reasoning effort defaults to `xhigh`.
