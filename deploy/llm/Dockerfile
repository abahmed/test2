# ── Stage 1: bake model ──
FROM --platform=$BUILDPLATFORM ollama/ollama AS bake

ARG MODEL_VERSION="v1"
ENV OLLAMA_MODELS=/models

COPY Modelfile /build/Modelfile

RUN ollama serve & \
    sleep 6 && \
    ollama pull granite4:1b-h && \
    ollama create kwatch-triage -f /build/Modelfile && \
    pkill ollama; sleep 1 && \
    # strip GPU runners HERE (before the COPY) so the image is genuinely slim —
    # a stage-2 rm would not shrink it (files persist in the COPY layer). CPU-only sidecar.
    rm -rf /usr/lib/ollama/cuda_* \
           /usr/lib/ollama/rocm* \
           /usr/lib/ollama/mlx* \
           /usr/lib/ollama/*cuda* \
           /usr/lib/ollama/*rocm* 2>/dev/null || true


# ── Stage 2: minimal runtime (CPU-only) ──
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

# copy ollama binary + CPU-only libs (GPU runners already stripped in bake stage)
COPY --from=bake /bin/ollama /bin/ollama
COPY --from=bake /usr/lib/ollama /usr/lib/ollama

# copy baked model
COPY --from=bake /models /models

# non-root user (safer for k8s)
RUN useradd -m -u 1000 nonroot && \
    chown -R 1000:1000 /models

USER 1000:1000

EXPOSE 11434

ENTRYPOINT ["/bin/ollama"]
CMD ["serve"]
