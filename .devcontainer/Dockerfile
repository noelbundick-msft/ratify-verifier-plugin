FROM mcr.microsoft.com/devcontainers/go:0-1-bullseye

# Install ORAS CLI: used to push plugins as OCI artifacts
ARG ORAS_VERSION=0.16.0
RUN curl -Lo oras.tar.gz https://github.com/oras-project/oras/releases/download/v${ORAS_VERSION}/oras_${ORAS_VERSION}_linux_amd64.tar.gz \
    && tar -zxf oras.tar.gz -C /usr/local/bin oras \
    && rm -rf oras.tar.gz
