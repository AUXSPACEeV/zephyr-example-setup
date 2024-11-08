#syntax=docker/dockerfile:1.0.0-experimental
#escape=\
#
# Auxspace e.V. build container for zephyr projects.
#
# Maintainer: Maximilian Stephan
#

# digests to FROM debian:trixie-slim
FROM debian@sha256:4bf4a3ee5cd9a4a6bc048af9bf7c0666f761e391e7ebb158b3da554ffe75994e

# Default User 10001:10001 - replace with `id -u`:`id -g` in container build
ARG PUID=10001
ARG PGID=10001
ARG ZEPHYR_SOURCE_SW_VERSION="main"

# Debian container fixes
ENV DEBIAN_FRONTEND=noninteractive

# Home directory for builder user
ENV BUILDER_HOME="/builder"

# Update APT index + setup and install locales
RUN apt-get update && \
    apt-get install -y --no-install-recommends locales && \
    localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

# Zephyr apt requirements
RUN apt-get install -y --no-install-recommends \
    git cmake ninja-build gperf \
    ccache dfu-util device-tree-compiler wget \
    python3-dev python3-pip python3-setuptools python3-tk python3-wheel python3-venv \
    xz-utils file make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1

# Other packages required in the container
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gosu sudo openssh-client

# Setup the container builder user
RUN groupadd --gid ${PGID} builder
RUN useradd --create-home \
    --shell /bin/bash \
    --home-dir "${BUILDER_HOME}" \
    --uid ${PUID} \
    --gid ${PGID} \
    --groups sudo,dialout,audio,video,disk \
    builder
RUN echo "builder  ALL=(ALL:ALL) NOPASSWD:ALL" > /etc/sudoers.d/builder-nopasswd
RUN echo 'Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy"' >> /etc/sudoers.d/builder-nopasswd

# Python virtualenv for pip packages
RUN mkdir -p /builder/.ssh && chmod 700 /builder/.ssh
RUN mkdir -p /opt/venv
RUN chown -R ${PUID}:${PGID} /opt/venv
RUN chown -R ${PUID}:${PGID} /builder
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install zephyr python dependencies + SDK
RUN python3 -m pip install west
RUN cd /builder && \
    wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.17.0/zephyr-sdk-0.17.0_linux-x86_64.tar.xz && \
    wget -O - https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.17.0/sha256.sum | shasum --check --ignore-missing && \
    tar xvf zephyr-sdk-0.17.0_linux-x86_64.tar.xz && \
    cd zephyr-sdk-0.17.0 && \
    ./setup.sh -c -h
# To install a certain type of sdk:
# ./setup.sh -c -h -t aarch64-zephyr-elf

# TODO: Integrate into west to be done at this point

# Download auxspace zephyr
RUN --mount=type=ssh west init \
    -m git@github.com:AUXSPACEeV/zephyr-example-setup.git \
    --mr ${ZEPHYR_SOURCE_SW_VERSION} \
    zephyr-workspace

# Set up zephyr
RUN --mount=type=ssh cd /builder/zephyr && \
    west update && \
    west zephyr-export && \
    pip install -r ./zephyr/scripts/requirements.txt

# Change permissions for the builder user
RUN chown -R ${PUID}:${PGID} /opt/venv
RUN chown -R ${PUID}:${PGID} /builder
RUN chown -R ${PUID}:${PGID} /builder/zephyr

# Configure user
USER builder
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

# Set and start the container entrypoint
COPY container-entrypoint.sh ~/container-entrypoint.sh

ENTRYPOINT [ "/builder/container-entrypoint.sh" ]
