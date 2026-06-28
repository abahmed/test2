# Shared Ollama version for BOTH the model-baker and the runtime binary, so the
# stage that runs `ollama create` and the stage that ships `/bin/ollama` are
# guaranteed identical across the amd64/arm64 split (no blob-format drift).
ARG OLLAMA_VERSION=latest

# ── Stage 1: bake model (arch-agnostic GGUF — built once on the native runner) ──
# Keep --platform=$BUILDPLATFORM here: this stage only produces /models, which is
# architecture-independent, and we want the slow `ollama serve/create` to run
# NATIVELY (not under QEMU emulation).
FROM --platform=$BUILDPLATFORM ollama/ollama:${OLLAMA_VERSION} AS bake

ENV OLLAMA_MODELS=/models

COPY Modelfile /build/Modelfile

RUN ollama serve & \
    sleep 6 && \
    ollama pull granite4:1b-h && \
    ollama create kwatch-triage -f /build/Modelfile && \
    pkill ollama


# ── Stage 2: per-target-arch ollama binary + CPU libs ──
# NO --platform pin → buildx resolves this to EACH target arch (amd64 AND arm64),
# so /bin/ollama and /usr/lib/ollama are the correct architecture. The only step
# here is `rm` (trivial under emulation). GPU runners stripped for the CPU sidecar.
FROM ollama/ollama:${OLLAMA_VERSION} AS ollamabin

RUN rm -rf /usr/lib/ollama/cuda_* \
           /usr/lib/ollama/rocm* \
           /usr/lib/ollama/mlx* \
           /usr/lib/ollama/*cuda* \
           /usr/lib/ollama/*rocm* 2>/dev/null || true


# ── Stage 3: minimal runtime (CPU-only) ──
FROM debian:bookworm-slim

ARG MODEL_VERSION="v1"

LABEL model.version="$MODEL_VERSION" \
      model.name="kwatch-triage" \
      model.base="granite4:1b-h" \
      description="Kwatch RCA model (CPU-only Ollama, Granite 4.0-H-1B)"

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates libgomp1 libstdc++6 && \
    rm -rf /var/lib/apt/lists/*

ENV OLLAMA_MODELS=/models \
    OLLAMA_HOST=0.0.0.0:11434

# correct-arch ollama binary + CPU-only libs (from the per-target-arch stage)
COPY --from=ollamabin /bin/ollama /bin/ollama
COPY --from=ollamabin /usr/lib/ollama /usr/lib/ollama

# arch-agnostic baked model (from the build-platform bake stage)
COPY --from=bake /models /models

# non-root user (safer for k8s)
RUN useradd -m -u 1000 nonroot && \
    chown -R 1000:1000 /models

USER 1000:1000

EXPOSE 11434

ENTRYPOINT ["/bin/ollama"]
CMD ["serve"]
