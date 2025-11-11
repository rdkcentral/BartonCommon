# ------------------------------ tabstop = 4 ----------------------------------
#
# If not stated otherwise in this file or this component's LICENSE file the
# following copyright and licenses apply:
#
# Copyright 2024 Comcast Cable Communications Management, LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
# SPDX-License-Identifier: Apache-2.0
#
# ------------------------------ tabstop = 4 ----------------------------------

#
# Created by Kevin Funderburg on 09/11/2024
#

###############################################################################
# Example workflow for building the image and pushing it to the GitHub Container
# Registry
#
# 1. Build the image:
#    `docker build -t ghcr.io/rdkcentral/bartoncommon_builder:some-tag .`
#
# 2. Authenticate with the GitHub Container Registry:
#    `echo $GITHUB_PAT | docker login ghcr.io -u $GITHUB_ACTOR --password-stdin`
#
#    where $GITHUB_PAT is your GitHub Personal Access Token and $GITHUB_ACTOR is
#    your GitHub username
#
# 3. Push the image to the GitHub Container Registry:
#    `docker push ghcr.io/rdkcentral/bartoncommon_builder:some-tag`
###############################################################################

###############################################################################
# base stage which provides a base image which has packages to support basic
# download functionality
###############################################################################
FROM ubuntu:24.04 AS base

# Delete the default ubuntu docker user: it can cause issues with user id 1000 on the host.
RUN touch /var/mail/ubuntu && chown ubuntu /var/mail/ubuntu && userdel -r ubuntu

RUN apt-get update && apt-get -y upgrade && DEBIAN_FRONTEND='noninteractive' apt-get install -y \
    build-essential \
    libarchive-tools \
    unzip \
    wget \
    zip \
    gcc-11 \
    g++-11 \
    python3-pip

# Set the default gcc and g++ versions to 11 because our current checkout of matter can't handle 13
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 30 && \
    update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 30 && \
    update-alternatives --install /usr/bin/gcov gcov /usr/bin/gcov-11 30 && \
    update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 20 && \
    update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-13 20 && \
    update-alternatives --install /usr/bin/gcov gcov /usr/bin/gcov-13 20

###############################################################################
# expands the base stage to allow for building of packages with common tools
###############################################################################
FROM base AS base_builder
RUN apt-get update && apt-get -y upgrade && DEBIAN_FRONTEND='noninteractive' apt-get install -y \
    autoconf \
    autoconf-archive \
    git \
    libtool \
    pkg-config \
    libsystemd-dev

# Create a symlink for the versioned dist-packages directory so the container can determine
# the correct PYTHONPATH at runtime
RUN bash -c "set -o pipefail && \
    PYTHON_VERSION=$(python3 --version | awk '{print $2}' | awk -F. '{print $1"."$2}') && \
    mkdir -p /usr/local/lib/python3.x && \
    ln -s /usr/local/lib/python${PYTHON_VERSION}/dist-packages /usr/local/lib/python3.x/dist-packages"

###############################################################################
# Third party build stage
#
# This stage is responsible for building specific third-party libraries that
# need to be versioned precisely for the Barton image build. Since apt cannot
# be relied upon to provide the correct versions, these libraries must be built
# from source.
###############################################################################
FROM base_builder AS third_party_builder

RUN apt-get update && apt-get -y upgrade && DEBIAN_FRONTEND='noninteractive' apt-get install -y \
    lsb-release \
    sudo \
    jq \
    cmake \
    libcjson-dev \
    libcmocka-dev \
    libssl-dev \
    libcurl4-openssl-dev \
    uuid-dev \
    libxml2-dev \
    libmbedtls-dev \
    libglib2.0-dev

# Download, install, and clean up OpenSSL 1.1.1v, specifically for use in our project
RUN cd /tmp && \
    wget https://github.com/openssl/openssl/releases/download/OpenSSL_1_1_1v/openssl-1.1.1v.tar.gz && \
    tar -xzvf openssl-1.1.1v.tar.gz && \
    cd openssl-1.1.1v && \
    ./config --prefix=/usr/local/openssl --openssldir=/usr/local/openssl && \
    make && \
    make install && \
    rm -rf /tmp/openssl-1.1.1v.tar.gz /tmp/openssl-1.1.1v
COPY openssl.conf /etc/ld.so.conf.d/
RUN ldconfig

# Download, build, and install libcurl, but link against OpenSSL 1.1.1v specifically
RUN cd /tmp && \
    wget https://curl.se/download/curl-7.82.0.tar.gz && \
    tar -xzvf curl-7.82.0.tar.gz && \
    cd curl-7.82.0 && \
    ./configure --with-ssl=/usr/local/openssl && \
    make && \
    make install && \
    rm -rf /tmp/curl-7.82.0.tar.gz /tmp/curl-7.82.0

RUN ls /tmp/

# Ensure the system uses OpenSSL 3 by default
ENV LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu:/usr/local/openssl/lib:$LD_LIBRARY_PATH

###############################################################################
# Development build stage
#
# This stage is responsible for installing packages and other tasks useful for
# development.
###############################################################################
FROM third_party_builder AS dev_builder

# Install packages useful for development
RUN apt-get update && apt-get -y upgrade && DEBIAN_FRONTEND='noninteractive' apt-get install -y \
    clangd \
    clang-format \
    silversearcher-ag \
    fd-find \
    gdb \
    rr \
    vim \
    pipx \
    gh \
    fish \
    strace \
    tcpdump \
    lcov \
    rsync \
    bash-completion \
    libcap2-bin

RUN pip3 install --upgrade --break-system-packages \
    stack-pr

# Make some /usr/ stuff open permissions. These are individual containers,
# so this will let people install barton outputs.
RUN chmod -R +777 /usr/local && \
    chmod +777 /usr/lib

# Set up pcap permissions group with a GID that is unlikely to conflict with any existing groups
RUN groupadd -g 10000 pcap && \
    chgrp pcap /usr/bin/tcpdump && \
    chmod 750 /usr/bin/tcpdump && \
    setcap cap_net_raw,cap_net_admin=eip /usr/bin/tcpdump

# Add bashrc_term_mods. We're looking to inject this before user creation/customization
COPY bashrc_term_mods /tmp/bashrc_term_mods

# This script will set up the user specific environment. It is purposefully not run yet,
# in favor of user creation at run time in the entrypoint script on the CLI, or in the extended
# Dockerfile.devcontainer to bake the user into the image for the devcontainer.
COPY setupUser.sh /setupUser.sh
RUN chmod +x /setupUser.sh

# Setup our entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["bash"]
