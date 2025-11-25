# =========================
# 1️⃣  Build Stage
# =========================
ARG RUST_VERSION=1.90
FROM rust:${RUST_VERSION} AS builder-rust
WORKDIR /app

# Install protoc
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev && \
    rm -rf /var/lib/apt/lists/*

# 💡 IMPROVEMENT:
# We use one large RUN command with mounts.
# This keeps the layer count low and maximizes cache reuse.

# Note on Mounts:
# 1. type=bind: Makes source code available read-only without copying it (fast).
# 2. type=cache: Persists 'target' and 'registry' between builds.

RUN --mount=type=bind,source=Cargo.toml,target=Cargo.toml \
    --mount=type=bind,source=Cargo.lock,target=Cargo.lock \
    --mount=type=bind,source=api,target=api \
    --mount=type=bind,source=clients/rust,target=clients/rust \
    --mount=type=bind,source=output-worker,target=output-worker \
    --mount=type=bind,source=sentry-integration,target=sentry-integration \
    --mount=type=bind,source=protobuf,target=protobuf \
    --mount=type=cache,sharing=locked,target=/app/target \
    --mount=type=cache,sharing=locked,target=/usr/local/cargo/registry \
    bash -euxo pipefail -c "\
        echo '🚀 Starting Rust build for API...'; \
        cd api; \
        echo '📦 Building with features: ${FEATURES:-default}'; \
        # SQLX_OFFLINE is crucial if you aren't connecting to a live DB during build \
        SQLX_OFFLINE=true cargo build --locked --release --no-default-features \
            --features '${FEATURES},hook0-client/reqwest-rustls-tls-webpki-roots'; \
        \
        echo '✅ Build finished, copying binary...'; \
        # We copy out of the cached target directory into the layer \
        cp ../target/release/hook0-api /hook0-api; \
        ls -lh /hook0-api; \
    "

# =========================
# 2️⃣  Runtime Stage
# =========================
FROM debian:12-slim
ARG UID=10001

# Create a non-privileged user
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser

# Install runtime dependencies (SSL certs are often needed for Rust apps making web requests)
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*

USER appuser

# Copy the binary from the builder
COPY --from=builder-rust /hook0-api /hook0-api

ENV DISABLE_SERVING_WEBAPP=true
CMD ["/hook0-api"]