FROM alpine:3.22.7 AS base

# Install dependencies
RUN apk add --no-cache openssl

# Make directory to store artifacts
RUN mkdir -p /flyteorg/bin /flyteorg/share

# Install k3s
ARG K3S_VERSION="v1.21.1%2Bk3s1"
ARG TARGETARCH

RUN case $TARGETARCH in \
    amd64) export SUFFIX=;; \
    arm64) export SUFFIX=-arm64;; \
    aarch64)  export SUFFIX=-arm64;; \
    # TODO: Check if we need to add case fail
    esac; \
    wget -q -O /flyteorg/bin/k3s https://github.com/k3s-io/k3s/releases/download/${K3S_VERSION}/k3s${SUFFIX} \
    && chmod +x /flyteorg/bin/k3s

# Install Helm
ARG HELM_VERSION="v3.6.3"

RUN wget -q -O /flyteorg/bin/get_helm.sh https://raw.githubusercontent.com/helm/helm/${HELM_VERSION}/scripts/get-helm-3 && \
    chmod 700 /flyteorg/bin/get_helm.sh && \
    sh /flyteorg/bin/get_helm.sh --version ${HELM_VERSION} && \
    mv /usr/local/bin/helm /flyteorg/bin/helm && \
    rm /flyteorg/bin/get_helm.sh


# Install flytectl
RUN wget -q -O - https://raw.githubusercontent.com/flyteorg/flytectl/master/install.sh | BINDIR=/flyteorg/bin sh -s

# Copy flyte chart
COPY charts/flyte/ /flyteorg/share/flyte
COPY charts/flyte-core/ /flyteorg/share/flyte-core
COPY charts/flyte-deps/ /flyteorg/share/flyte-deps

# Copy scripts
COPY docker/sandbox/kubectl docker/sandbox/cgroup-v2-hack.sh docker/sandbox/wait-for-flyte.sh /flyteorg/bin/

# Docker engine comes from the Alpine 3.22 repo (28.3.x, still patched by Alpine)
# instead of the docker:28.x-dind image, which is no longer rebuilt. Docker 29
# raises the minimum API version to 1.44 and would break the k3s v1.21
# dockershim client, so we stay on 28. The entrypoint runs dockerd directly.
FROM alpine:3.22.7 AS dind

# Install dependencies (docker engine/cli plus what docker:dind ships for dockerd)
RUN apk add --no-cache docker iptables ip6tables e2fsprogs xfsprogs pigz xz openssl ca-certificates \
    bash git make tini curl jq

# Copy artifacts from base
COPY --from=base /flyteorg/ /flyteorg/

# Copy entrypoints
COPY docker/sandbox/flyte-entrypoint-default.sh /flyteorg/bin/flyte-entrypoint.sh

ARG FLYTE_VERSION="latest"
ENV FLYTE_VERSION="${FLYTE_VERSION}"

ARG FLYTE_TEST="release"
ENV FLYTE_TEST="${FLYTE_TEST}"

# Update PATH variable
ENV PATH="/flyteorg/bin:${PATH}"

# Declare volumes for dockerd (the docker:dind image declared this) and k3s
VOLUME /var/lib/docker
VOLUME /var/lib/kubelet
VOLUME /var/lib/rancher/k3s
VOLUME /var/lib/cni
VOLUME /var/log

# Expose Flyte ports
EXPOSE 30081 30082 30084 30088


ENTRYPOINT ["tini", "flyte-entrypoint.sh"]
