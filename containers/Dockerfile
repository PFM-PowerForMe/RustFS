FROM ghcr.io/pfm-powerforme/rustfs-frontend:latest AS frontend


FROM docker.io/library/alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /
RUN git -c advice.detachedHead=false clone --branch $TAG --depth=1 --recurse-submodules https://github.com/pfm-powerforme/rustfs.git source
WORKDIR /source/
RUN rm -rf rustfs/static/
COPY --from=frontend /frontend/ /source/rustfs/static/


FROM docker.io/library/rust:alpine AS backend
RUN rustup component add rustfmt
RUN apk update && apk add --no-cache \
	musl-dev \
	build-base \
	git \
	mold
RUN apk add flatbuffers-dev --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community
RUN apk add protobuf-dev --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community
RUN flatc --version
RUN protoc --version
WORKDIR /
COPY --from=git /source/ /backend/
WORKDIR /backend/
RUN mkdir -p /backend/target/
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    cargo fetch --locked || true
ENV CARGO_NET_GIT_FETCH_WITH_CLI=true \
    CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_RELEASE_DEBUG=false \
    CARGO_PROFILE_RELEASE_SPLIT_DEBUGINFO=off \
    CARGO_PROFILE_RELEASE_STRIP=symbols \
    CARGO_TARGET_DIR=/backend/target
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/backend/target \
    cargo run --bin gproto
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/backend/target \
    set -eux; \
    cargo build --profile production --locked --bin rustfs -j "$(nproc)"; \
    install -m 0755 target/production/rustfs /usr/local/bin/rustfs


FROM docker.io/library/alpine:latest AS runtime
RUN apk update && apk add --no-cache \
    ca-certificates bash tzdata coreutils gosu shadow libgcc \
    && update-ca-certificates
RUN addgroup -g 1000 rustfs && \
    adduser -u 1000 -G rustfs -s /usr/sbin/nologin -D rustfs
COPY --from=backend /usr/local/bin/rustfs /usr/bin/rustfs
COPY --from=git /source/entrypoint.sh /entrypoint.sh
RUN chmod +x /usr/bin/rustfs /entrypoint.sh && \
    mkdir -p /data /logs && \
    chown rustfs:rustfs /data /logs && \
    chmod 750 /data /logs
ENV RUSTFS_ADDRESS=:9000 \
    RUSTFS_ACCESS_KEY="rustfsadmin" \
    RUSTFS_SECRET_KEY="rustfsadmin" \
    RUSTFS_CONSOLE_ENABLE="true" \
    RUSTFS_VOLUMES="/data" \
    RUST_LOG="warn" \
    RUSTFS_OBS_LOG_DIRECTORY="/logs" \
    RUSTFS_SINKS_FILE_PATH="/logs" \
    RUSTFS_USERNAME="rustfs" \
    RUSTFS_GROUPNAME="rustfs" \
    RUSTFS_UID="1000" \
    RUSTFS_GID="1000"
EXPOSE 9000
VOLUME ["/data", "/logs"]
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/bin/rustfs"]
