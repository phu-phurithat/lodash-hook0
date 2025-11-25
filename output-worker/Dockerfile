# =========================
# 1️⃣  Dependencies Stage
# =========================
ARG RUST_VERSION=1.90
FROM rust:${RUST_VERSION} AS deps
WORKDIR /app

# Install protoc for prost/protobuf build scripts
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev && \
    rm -rf /var/lib/apt/lists/*

# Copy dependency manifests
COPY Cargo.toml Cargo.lock ./

# Copy source directories for dependencies
COPY api/Cargo.toml api/
COPY clients/rust/Cargo.toml clients/rust/
COPY output-worker/Cargo.toml output-worker/
COPY sentry-integration/Cargo.toml sentry-integration/
COPY protobuf/Cargo.toml protobuf/

# Create dummy main files to build dependencies
RUN mkdir -p api/src clients/rust/src output-worker/src sentry-integration/src protobuf/src && \
    echo "fn main() {}" > api/src/main.rs && \
    echo "fn main() {}" > clients/rust/src/lib.rs && \
    echo "fn main() {}" > output-worker/src/main.rs && \
    echo "fn main() {}" > sentry-integration/src/lib.rs && \
    echo "fn main() {}" > protobuf/src/lib.rs

# Build dependencies only
RUN --mount=type=cache,sharing=private,target=/usr/local/cargo/registry \
    cargo build --release && \
    rm -rf target/release/deps/hook0-* target/release/deps/output-*

# =========================
# 2️⃣  Build Stage
# =========================
FROM deps AS builder-rust
WORKDIR /app

# Copy source code
COPY api/ api/
COPY clients/rust/ clients/rust/
COPY output-worker/ output-worker/
COPY sentry-integration/ sentry-integration/
COPY protobuf/ protobuf/

# Build hook0-output-worker using BuildKit mounts
RUN --mount=type=cache,sharing=private,target=/app/target \
    --mount=type=cache,sharing=private,target=/usr/local/cargo/registry \
    bash -c "set -e; cd output-worker; SQLX_OFFLINE=true cargo build --locked --release; cp ../target/release/hook0-output-worker /"

# =========================
# 2️⃣  Runtime Stage
# =========================
FROM debian:12-slim
ARG UID=10001

# Create a non-root user for security
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser

USER appuser

# Copy binary from builder
COPY --from=builder-rust /hook0-output-worker /hook0-output-worker

# Optional environment variable for runtime behavior
ENV DISABLE_SERVING_WEBAPP=true

# Default entrypoint
CMD ["/hook0-output-worker"]
