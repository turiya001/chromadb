FROM rust:1.81.0 AS builder

ARG RELEASE_MODE=

WORKDIR /chroma/

ENV PROTOC_ZIP=protoc-25.1-linux-x86_64.zip
RUN curl -OL https://github.com/protocolbuffers/protobuf/releases/download/v25.1/$PROTOC_ZIP \
    && unzip -o $PROTOC_ZIP -d /usr/local bin/protoc \
    && unzip -o $PROTOC_ZIP -d /usr/local 'include/*' \
    && rm -f $PROTOC_ZIP

# Install python for pyo3
RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*

COPY Cargo.toml Cargo.toml
COPY Cargo.lock Cargo.lock
COPY idl/ idl/
COPY rust/ rust/

FROM builder AS query_service_builder
# sharing=locked is necessary to prevent cargo build from running concurrently on the same mounted directory
RUN --mount=type=cache,sharing=locked,target=/chroma/target/ \
    --mount=type=cache,sharing=locked,target=/usr/local/cargo/registry/ \
    if [ "$RELEASE_MODE" = "1" ]; then cargo build --bin query_service --release; else cargo build --bin query_service; fi && \
    if [ "$RELEASE_MODE" = "1" ]; then mv target/release/query_service ./query_service; else mv target/debug/query_service ./query_service; fi

FROM builder AS compaction_service_builder
RUN --mount=type=cache,sharing=locked,target=/chroma/target/ \
    --mount=type=cache,sharing=locked,target=/usr/local/cargo/registry/ \
    if [ "$RELEASE_MODE" = "1" ]; then cargo build --bin compaction_service --release; else cargo build --bin compaction_service; fi && \
    if [ "$RELEASE_MODE" = "1" ]; then mv target/release/compaction_service ./compaction_service; else mv target/debug/compaction_service ./compaction_service; fi


FROM debian:bookworm-slim AS runner
RUN apt-get update && apt-get install -y libssl-dev ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /chroma/rust/worker/chroma_config.yaml .

# Install python for pyo3
RUN apt-get update && apt-get install -y python3 python3-pip && rm -rf /var/lib/apt/lists/*


FROM runner AS query_service
COPY --from=query_service_builder /chroma/query_service .
ENTRYPOINT [ "./query_service" ]

FROM runner AS compaction_service
COPY --from=compaction_service_builder /chroma/compaction_service .
ENTRYPOINT [ "./compaction_service" ]
