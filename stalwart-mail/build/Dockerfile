FROM debian:buster-slim AS chef
RUN apt-get update && \
    export DEBIAN_FRONTEND=noninteractive && \
    apt-get install -yq \
    build-essential \
    cmake \
    clang \
    curl \
    protobuf-compiler
ENV RUSTUP_HOME=/opt/rust/rustup \
    PATH=/home/root/.cargo/bin:/opt/rust/cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
RUN curl https://sh.rustup.rs -sSf | \
    env CARGO_HOME=/opt/rust/cargo \
    sh -s -- -y --default-toolchain stable --profile minimal --no-modify-path && \
    env CARGO_HOME=/opt/rust/cargo \
    rustup component add rustfmt
RUN curl -LO https://github.com/apple/foundationdb/releases/download/7.3.62/foundationdb-clients_7.3.62-1_amd64.deb && \
    dpkg -i foundationdb-clients_7.3.62-1_amd64.deb
RUN env CARGO_HOME=/opt/rust/cargo cargo install cargo-chef && \
    rm -rf /opt/rust/cargo/registry/
WORKDIR /app

FROM chef AS planner
COPY Cargo.toml .
COPY Cargo.lock .
COPY crates/ crates/
COPY resources/ resources/
COPY tests/ tests/
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY Cargo.toml .
COPY Cargo.lock .
COPY crates/ crates/
COPY resources/ resources/
COPY tests/ tests/
RUN cargo build --manifest-path=crates/main/Cargo.toml --no-default-features --features "sqlite postgres mysql rocks elastic s3 redis azure enterprise foundationdb" --release

FROM debian:buster-slim AS runtime

COPY --from=builder /app/target/release/stalwart-mail /usr/local/bin/stalwart-mail
RUN apt-get update -y && apt-get install -yq ca-certificates curl
RUN curl -LO https://github.com/apple/foundationdb/releases/download/7.3.62/foundationdb-clients_7.3.62-1_amd64.deb && \
    dpkg -i foundationdb-clients_7.3.62-1_amd64.deb
RUN useradd stalwart-mail -s /sbin/nologin -M
RUN mkdir -p /opt/stalwart-mail
RUN chown stalwart-mail:stalwart-mail /opt/stalwart-mail

ENTRYPOINT ["/usr/local/bin/stalwart-mail", "--config", "/opt/stalwart-mail/etc/config.toml"]
