# =========================
# 1️⃣  Build Stage
# =========================
ARG RUST_VERSION=1.90
FROM rust:${RUST_VERSION} AS builder-rust
WORKDIR /app

# Install protoc for prost/protobuf crates
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends protobuf-compiler libprotobuf-dev && \
    rm -rf /var/lib/apt/lists/*

# Use BuildKit mounts for cache
RUN --mount=type=bind,source=Cargo.toml,target=Cargo.toml \
    --mount=type=bind,source=Cargo.lock,target=Cargo.lock \
    --mount=type=bind,source=api,target=api \
    --mount=type=bind,source=clients/rust,target=clients/rust \
    --mount=type=bind,source=output-worker,target=output-worker \
    --mount=type=bind,source=sentry-integration,target=sentry-integration \
    --mount=type=bind,source=protobuf,target=protobuf \
    --mount=type=cache,sharing=private,target=/app/target \
    --mount=type=cache,sharing=private,target=/usr/local/cargo/registry \
    bash -euxo pipefail -c "\
        echo '🚀 Starting Rust build for API...'; \
        cd api; \
        echo '📦 Building with features: ${FEATURES}'; \
        SQLX_OFFLINE=true cargo build --locked --release --no-default-features \
            --features '${FEATURES},hook0-client/reqwest-rustls-tls-webpki-roots'; \
        echo '✅ Build finished, copying binary...'; \
        cp ../target/release/hook0-api /; \
        ls -lh /hook0-api; \
        echo '🎯 Rust API binary ready.' \
    "
# =========================
# 2️⃣  Runtime Stage
# =========================
FROM debian:12-slim
ARG UID=10001

RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser

USER appuser
COPY --from=builder-rust /hook0-api /hook0-api

ENV DISABLE_SERVING_WEBAPP=true
CMD ["/hook0-api"]
