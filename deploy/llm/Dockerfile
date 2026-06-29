# syntax=docker/dockerfile:1.7

# ---------- Stage 1: download ----------
FROM --platform=$BUILDPLATFORM debian:bookworm-slim AS download

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        tar && \
    rm -rf /var/lib/apt/lists/*

ARG TARGETARCH
ARG LLAMACPP_TAG="b5098"
ARG LLAMACPP_BUILD="5098"

RUN set -eux; \
    case "$TARGETARCH" in \
        amd64) ASSET="ubuntu-x64" ;; \
        arm64) ASSET="ubuntu-arm64" ;; \
        *) echo "Unsupported architecture: $TARGETARCH"; exit 1 ;; \
    esac; \
    URL="https://github.com/ggml-org/llama.cpp/releases/download/${LLAMACPP_TAG}/llama-b${LLAMACPP_BUILD}-bin-${ASSET}.tar.gz"; \
    echo "Downloading $URL"; \
    curl -fL "$URL" -o /tmp/llama.tar.gz; \
    mkdir -p /tmp/llama; \
    tar -xzf /tmp/llama.tar.gz -C /tmp/llama; \
    cp "$(find /tmp/llama -type f -name llama-server | head -n1)" /usr/local/bin/llama-server; \
    chmod +x /usr/local/bin/llama-server

RUN mkdir -p /models && \
    curl -fL \
      "https://huggingface.co/openbmb/MiniCPM5-1B-GGUF/resolve/main/MiniCPM5-1B-Q4_K_M.gguf" \
      -o /models/model.gguf

# ---------- Stage 2: runtime ----------
FROM debian:bookworm-slim

ARG MODEL_VERSION="v1"

LABEL model.version="${MODEL_VERSION}" \
      model.name="kwatch-triage" \
      model.base="MiniCPM5-1B-Q4_K_M" \
      description="kwatch-triage over MiniCPM5-1B using llama.cpp"

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libstdc++6 \
        libgomp1 \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/* && \
    useradd -m -u 1000 nonroot

COPY --from=download /usr/local/bin/llama-server /usr/local/bin/llama-server
COPY --from=download /models/model.gguf /models/model.gguf

USER nonroot

EXPOSE 8080

ENTRYPOINT ["/usr/local/bin/llama-server"]

CMD ["--host","0.0.0.0","--port","8080","--model","/models/model.gguf","--threads","1","--ctx-size","8192","--jinja","--temp","0.7","--top-p","0.95","--repeat-penalty","1.1","--n-predict","256"]
