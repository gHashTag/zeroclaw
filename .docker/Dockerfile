# syntax=docker/dockerfile:1.7

# ── Stage 0: Frontend build ─────────────────────────────────────
FROM node:22-alpine AS web-builder
WORKDIR /web
COPY web/package.json web/package-lock.json* ./
RUN npm ci --ignore-scripts 2>/dev/null || npm install --ignore-scripts
COPY web/ .
RUN npm run build

# ── Stage 1: Build ────────────────────────────────────────────
FROM rust:1.94-slim@sha256:da9dab7a6b8dd428e71718402e97207bb3e54167d37b5708616050b1e8f60ed AS builder

WORKDIR /app
ARG ZEROCLAW_CARGO_FEATURES=""

# Install build dependencies
RUN apt-get update && apt-get install -y \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Copy workspace root - REQUIRED for workspace inheritance
COPY Cargo.toml Cargo.lock ./

# Copy workspace members to their correct paths (NOT flattened into /app)
COPY crates/aardvark-sys/Cargo.toml crates/aardvark-sys/build.rs crates/aardvark-sys/
COPY crates/robot-kit/Cargo.toml crates/robot-kit/
COPY crates/zeroclaw-api/Cargo.toml crates/zeroclaw-api/
COPY crates/zeroclaw-channels/Cargo.toml crates/zeroclaw-channels/
COPY crates/zeroclaw-config/Cargo.toml crates/zeroclaw-config/
COPY crates/zeroclaw-gateway/Cargo.toml crates/zeroclaw-gateway/
COPY crates/zeroclaw-hardware/Cargo.toml crates/zeroclaw-hardware/
COPY crates/zeroclaw-infra/Cargo.toml crates/zeroclaw-infra/
COPY crates/zeroclaw-macros/Cargo.toml crates/zeroclaw-macros/
COPY crates/zeroclaw-memory/Cargo.toml crates/zeroclaw-memory/
COPY crates/zeroclaw-plugins/Cargo.toml crates/zeroclaw-plugins/
COPY crates/zeroclaw-providers/Cargo.toml crates/zeroclaw-providers/
COPY crates/zeroclaw-runtime/Cargo.toml crates/zeroclaw-runtime/
COPY crates/zeroclaw-tool-call-parser/Cargo.toml crates/zeroclaw-tool-call-parser/
COPY crates/zeroclaw-tools/Cargo.toml crates/zeroclaw-tools/
COPY crates/zeroclaw-tui/Cargo.toml crates/zeroclaw-tui/
COPY apps/tauri/Cargo.toml apps/tauri/

# Create dummy targets in correct workspace structure
RUN mkdir -p src benches apps/tauri/src \
    && echo "fn main() {}" > src/main.rs \
    && echo "" > src/lib.rs \
    && echo "fn main() {}" > benches/agent_benchmarks.rs \
    && echo "fn main() {}" > apps/tauri/src/main.rs \
    && echo "fn main() {}" > apps/tauri/build.rs \
    && for d in crates/*/; do mkdir -p "${d}src" && printf '' > "${d}src/lib.rs"; done

# Build
RUN cargo build --release --locked

# Copy source code
COPY src/ src/
COPY benches/ benches/
COPY *.rs .
RUN touch src/main.rs
RUN cargo build --release --locked && \
    cp target/release/zeroclaw /app/zeroclaw && \
    strip /app/zeroclaw
RUN size=$(stat -c%s /app/zeroclaw) && \
    if [ "$size" -lt 1000000 ]; then echo "ERROR: binary too small (${size} bytes), likely dummy build artifact" && exit 1; fi

# Prepare runtime directory structure
RUN mkdir -p /zeroclaw-data/.zeroclaw /zeroclaw-data/workspace && \
    printf '%s\n' \
        'workspace_dir = "/zeroclaw-data/workspace"' \
        'config_path = "/zeroclaw-data/.zeroclaw/config.toml"' \
        'api_key = ""' \
        'default_provider = "openrouter"' \
        'default_model = "anthropic/claude-sonnet-4-20250514"' \
        'default_temperature = 0.7' \
        '' \
        '[gateway]' \
        'port = 42617' \
        'host = "[::]"' \
        'allow_public_bind = true' \
        'require_pairing = false' \
        'web_dist_dir = "/zeroclaw-data/web/dist"' \
        '' \
        '[autonomy]' \
        'level = "supervised"' \
        'auto_approve = ["file_read", "file_write", "file_edit", "memory_recall", "memory_store", "web_search_tool", "web_fetch", "calculator", "glob_search", "content_search", "image_info", "weather", "git_operations"]' \
        > /zeroclaw-data/.zeroclaw/config.toml && \
    chown -R 65534:65534 /zeroclaw-data

# ── Stage 2: Production Runtime (Distroless) ─────────────────
FROM gcr.io/distroless/cc-debian13:nonroot@sha256:84fcd3c223b144b0cb6edc5ecc75641819842a9679a3a58fd6294bec47532bf7 AS release

COPY --from=builder /app/zeroclaw /usr/local/bin/zeroclaw
COPY --from=builder /zeroclaw-data /zeroclaw-data
COPY --from=web-builder /web/dist /zeroclaw-data/web/dist

ENV LANG=C.UTF-8
ENV ZEROCLAW_WORKSPACE=/zeroclaw-data/workspace
ENV HOME=/zeroclaw-data
ENV ZEROCLAW_GATEWAY_PORT=42617

WORKDIR /zeroclaw-data
USER 65534:65534
EXPOSE 42617
HEALTHCHECK --interval=60s --timeout=10s --retries=3 --start-period=10s \
    CMD ["zeroclaw", "status", "--format=exit-code"]
ENTRYPOINT ["zeroclaw"]
CMD ["daemon"]
