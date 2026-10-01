FROM mcr.microsoft.com/devcontainers/rust:latest

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        gcc-riscv64-linux-gnu \
        qemu-user \
        jq \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN rustup target add riscv64gc-unknown-linux-gnu
