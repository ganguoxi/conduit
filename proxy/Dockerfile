# Proxy build and runtime
#
# When PROXY_UNOPTIMIZED is set and not empty, unoptimized rust artifacts are produced.
# This reduces build time and produces binaries with debug symbols, at the expense of
# runtime perforamnce.

ARG RUST_IMAGE=rust:1.26.2
ARG RUNTIME_IMAGE=gcr.io/runconduit/base:2017-10-30.01

## Builds the proxy as incrementally as possible.
FROM $RUST_IMAGE as build

WORKDIR /usr/src/conduit

# Fetch external dependencies.
#
# Mock out all local code and fetch external dependencies to ensure that external sources
# are cached.
RUN for d in proxy proxy/controller-grpc proxy/futures-mpsc-lossy proxy/router ; \
    do mkdir -p "${d}/src" && touch "${d}/src/lib.rs" ; \
    done
COPY Cargo.toml Cargo.lock ./
COPY proxy/Cargo.toml                       proxy/Cargo.toml
COPY proxy/controller-grpc/Cargo.toml       proxy/controller-grpc/Cargo.toml
COPY proxy/futures-mpsc-lossy/Cargo.toml    proxy/futures-mpsc-lossy/Cargo.toml
COPY proxy/router/Cargo.toml                proxy/router/Cargo.toml
RUN cargo fetch --locked

# Build libraries, leaving the proxy and gRPC bindings mocked out.
COPY proxy/futures-mpsc-lossy  proxy/futures-mpsc-lossy
COPY proxy/router              proxy/router
ARG PROXY_UNOPTIMIZED
RUN if [ -n "$PROXY_UNOPTIMIZED" ]; \
    then cargo build --frozen ; \
    else cargo build --frozen --release ; \
    fi

# Build gRPC bindings, leaving the proxy mocked out.
COPY proto/proxy proto/proxy
COPY proto/common.proto proto/common.proto
COPY proxy/controller-grpc proxy/controller-grpc
RUN if [ -n "$PROXY_UNOPTIMIZED" ]; \
    then cargo build -p conduit-proxy-controller-grpc --frozen ; \
    else cargo build -p conduit-proxy-controller-grpc --frozen --release ; \
    fi

# Build the proxy binary using pre-built dependencies.
COPY proxy/src      proxy/src
COPY proxy/tests    proxy/tests
RUN if [ -n "$PROXY_UNOPTIMIZED" ]; \
    then cargo build -p conduit-proxy --bin conduit-proxy --frozen ; \
    else cargo build -p conduit-proxy --bin conduit-proxy --frozen --release ; \
    fi

RUN if [ -n "$PROXY_UNOPTIMIZED" ]; \
    then mv target/debug/conduit-proxy   target/conduit-proxy ; \
    else mv target/release/conduit-proxy target/conduit-proxy ; \
    fi

## Install the proxy binary into the base runtime image.
FROM $RUNTIME_IMAGE as runtime
COPY --from=build /usr/src/conduit/target/conduit-proxy /usr/local/bin/conduit-proxy
ENV CONDUIT_PROXY_LOG=warn,conduit_proxy=info
ENTRYPOINT ["/usr/local/bin/conduit-proxy"]
