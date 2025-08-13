FROM docker.io/library/alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /
RUN git -c advice.detachedHead=false clone --branch $TAG --depth=1 --recurse-submodules https://github.com/pfm-powerforme/rustfs.git source
WORKDIR /source/


FROM ghcr.io/pfm-powerforme/rustfs-frontend:latest AS frontend


FROM docker.io/library/rust:alpine AS backend
RUN rustup component add rustfmt
RUN apk update && apk add --no-cache \
	musl-dev \
	build-base \
	git
RUN apk add flatbuffers-dev --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community
RUN apk add protobuf-dev --repository=https://dl-cdn.alpinelinux.org/alpine/edge/community
RUN flatc --version
RUN protoc --version
WORKDIR /
COPY --from=git /source/ /backend/
COPY --from=frontend /frontend/* /backend/rustfs/static/
WORKDIR /backend/
ENV CARGO_NET_GIT_FETCH_WITH_CLI=true \
    CARGO_REGISTRIES_CRATES_IO_PROTOCOL=sparse \
    CARGO_INCREMENTAL=0 \
    CARGO_PROFILE_RELEASE_DEBUG=false \
    CARGO_PROFILE_RELEASE_SPLIT_DEBUGINFO=off \
    CARGO_PROFILE_RELEASE_STRIP=true \
    RUSTFLAGS="-C target-feature=-crt-static"
RUN cargo run --bin gproto
RUN touch rustfs/build.rs
RUN cargo build --release -p rustfs --bins


FROM docker.io/library/alpine:latest AS runtime
RUN apk update && apk add --no-cache \
    ca-certificates bash gosu coreutils shadow libgcc \
    && update-ca-certificates
RUN addgroup -g 1000 rustfs && \
    adduser -u 1000 -G rustfs -s /bin/bash -D rustfs
COPY --from=backend /backend/target/release/rustfs /usr/bin/rustfs
COPY --from=git /source/entrypoint.sh /entrypoint.sh
RUN chmod +x /usr/bin/rustfs /entrypoint.sh && \
    mkdir -p /data /logs && \
    chown rustfs:rustfs /data /logs && \
    chmod 700 /data /logs
ENV RUSTFS_ADDRESS=:9000 \
    RUSTFS_ACCESS_KEY=rustfsadmin \
    RUSTFS_SECRET_KEY=rustfsadmin \
    RUSTFS_CONSOLE_ENABLE=true \
    RUSTFS_VOLUMES=/data \
    RUST_LOG=warn \
    RUSTFS_OBS_LOG_DIRECTORY=/logs \
    RUSTFS_SINKS_FILE_PATH=/logs
EXPOSE 9000
VOLUME ["/data", "/logs"]
ENTRYPOINT ["/entrypoint.sh"]
CMD ["/usr/bin/rustfs"]


