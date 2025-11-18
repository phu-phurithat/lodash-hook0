# =========================
# 1️⃣  Build Stage
# =========================
ARG RUST_VERSION=1.90
FROM rust:${RUST_VERSION} AS builder-rust
WORKDIR /app

# Install protoc for prost/protobuf build scripts
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev && \
    rm -rf /var/lib/apt/lists/*

# Build hook0-output-worker using BuildKit mounts
RUN --mount=type=bind,source=Cargo.toml,target=Cargo.toml \
    --mount=type=bind,source=Cargo.lock,target=Cargo.lock \
    --mount=type=bind,source=api,target=api \
    --mount=type=bind,source=clients/rust,target=clients/rust \
    --mount=type=bind,source=output-worker,target=output-worker \
    --mount=type=bind,source=sentry-integration,target=sentry-integration \
    --mount=type=bind,source=protobuf,target=protobuf \
    --mount=type=cache,sharing=private,target=/app/target \
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
