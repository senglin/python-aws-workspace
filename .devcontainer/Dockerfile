#-------------------------------------------------------------------------------------------------------------
# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License. See https://go.microsoft.com/fwlink/?linkid=2090316 for license information.
#-------------------------------------------------------------------------------------------------------------

FROM golang:1 AS fargatedist

# Avoid warnings by switching to noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# Configure apt, install packages and tools
RUN apt-get update \
    && apt-get -y install --no-install-recommends apt-utils dialog 2>&1 \
    #
    # Verify git, process tools, lsb-release (common in install instructions for CLIs) installed
    && apt-get -y install git openssh-client zip vim less iproute2 procps lsb-release upx file make \
    #
    # SSL dependencies
    && apt-get -y install apt-transport-https ca-certificates gnupg2 \
    #
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/gotools

WORKDIR /fargate

RUN git clone -b combined-fixes https://github.com/senglin/fargatecli.git .
RUN go mod download
RUN GOOS=linux GOARCH=amd64 go build -o fargate main.go

# Switch back to dialog for any ad-hoc use of apt-get
ENV DEBIAN_FRONTEND=dialog

#-------------------------------------------------------------------------------------------------------------


# Update the VARIANT arg in devcontainer.json to pick a Python version: 3, 3.8, 3.7, 3.6 
# To fully customize the contents of this image, use the following Dockerfile instead:
# https://github.com/microsoft/vscode-dev-containers/tree/v0.128.0/containers/python-3/.devcontainer/base.Dockerfile
# ARG VARIANT="3"
FROM mcr.microsoft.com/vscode/devcontainers/python:0-3.8

# This Dockerfile adds a non-root user with sudo access. Use the "remoteUser"
# property in devcontainer.json to use it. On Linux, the container user's GID/UIDs
# will be updated to match your local UID/GID (when using the dockerFile property).
# See https://aka.ms/vscode-remote/containers/non-root-user for details.
ARG USERNAME=vscode
# For Fargate CLI
ENV VISUAL=vim
ENV EDITOR=vim

# [Optional] If your requirements rarely change, uncomment this section to add them to the image.
#
# COPY requirements.txt /tmp/pip-tmp/
# RUN pip3 --disable-pip-version-check --no-cache-dir install -r /tmp/pip-tmp/requirements.txt \
#    && rm -rf /tmp/pip-tmp

# [Optional] Uncomment this section to install additional packages.
#
RUN apt-get update \
    && export DEBIAN_FRONTEND=noninteractive \
    && apt-get -y install --no-install-recommends vim


# AWS Tools
RUN \
    # Install AWS CLI v2: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && sudo ./aws/install \
    && rm -rf aws awscliv2.zip \
    #
    # Install AWS SAM CLI
    # Install Homebrew before installing aws-sam-cli
    && apt-get -y install ruby build-essential groff \
    && echo | sudo -u ${USERNAME} sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)" \
    && echo 'eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv)' >> /home/${USERNAME}/.bashrc \
    && eval $(/home/linuxbrew/.linuxbrew/bin/brew shellenv) \
    && brew tap aws/tap \
    && brew install aws-sam-cli \
    #
    # Install AWS ECS CLI
    && sudo curl -o /usr/local/bin/ecs-cli https://s3.cn-north-1.amazonaws.com.cn/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && sudo chmod +x /usr/local/bin/ecs-cli
    #
    # Install AWS AMPLIFY CLI
    RUN curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash - \
    && apt-get install -y nodejs \
    && npm install -g @aws-amplify/cli
    #
    # Install FARGATE CLI
    COPY --from=fargatedist /fargate/fargate /usr/local/bin/
    # && curl -L -o fargate-0.3.2-linux-amd64.zip https://github.com/awslabs/fargatecli/releases/download/0.3.2/fargate-0.3.2-linux-amd64.zip \
    # && sudo unzip fargate-0.3.2-linux-amd64.zip -d /usr/local/bin/ \
    # && rm -rf fargate-0.3.2-linux-amd64.zip
    #
    # Install script to automate the emptying of AWS S3 bucket (with object versioning enabled)
    COPY aws-s3-empty-bucket.sh /usr/local/bin/

# Docker installation
RUN \
    # Install Docker CE CLI (and engine)
    # Fix to version due to bug in 19.03.11 (https://github.com/docker/for-linux/issues/1033)
    apt-get install -y apt-transport-https ca-certificates curl gnupg2 lsb-release \
    && curl -fsSL https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/gpg | apt-key add - 2>/dev/null \
    && echo "deb [arch=amd64] https://download.docker.com/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]') $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y containerd.io socat docker-ce-cli=5:19.03.10\* docker-ce=5:19.03.10\* \
    #
    # Install Docker Compose
    && LATEST_COMPOSE_VERSION=$(curl -sSL "https://api.github.com/repos/docker/compose/releases/latest" | grep -o -P '(?<="tag_name": ").+(?=")') \
    && curl -sSL "https://github.com/docker/compose/releases/download/${LATEST_COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose \
    #
    # Fix to run docker in Debian 10 (Buster)
    # https://forums.docker.com/t/failing-to-start-dockerd-failed-to-create-nat-chain-docker/78269
    && update-alternatives --set iptables /usr/sbin/iptables-legacy \
    && update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy


# 3-WAYS TO ENABLE DOCKER IN THE CONTAINER
# https://github.com/Microsoft/vscode-dev-containers/tree/master/containers/docker-in-docker
RUN echo '#!/bin/sh\n\
    if [ "$VSC_DIND" = "root" ] \n\
    then \n\
        # Create docker-root-entrypoint.sh to link host docker socket to container docker socket. This is only for root access.
        # In order to use this script, please ensure that remoteUser is set to root (or removed) in devcontainer.json file.
        sudo ln -s /var/run/docker-host.sock /var/run/docker.sock \n\
        echo Symbolic created at /var/run/docker.sock \n\
    elif [ "$VSC_DIND" = "nonroot" ] \n\
    then \n\
        # Create docker-nonroot-entrypoint.sh to spin up socat, to enable non-root access
        sudo rm -rf /var/run/docker.sock\n\
        ((sudo socat UNIX-LISTEN:/var/run/docker.sock,fork,mode=660,user='${USERNAME}' UNIX-CONNECT:/var/run/docker-host.sock) >> /tmp/vscr-dind-socat.log 2>&1) & > /dev/null\n\
        echo socat: UNIX-LISTEN:/var/run/docker.sock UNIX-CONNECT:/var/run/docker-host.sock \n\
    else \n\
        # Create dockerd-entrypoint.sh to start dockerd in the container, please make sure that non-root users are added to docker group
        # This requires --privileged to be set on devcontainer.json
        ((sudo dockerd --host="unix:///var/run/docker.sock") >> /tmp/vscr-dockerd.log 2>&1) & >/dev/null\n\
        sudo usermod -aG docker '${USERNAME}' \n\
        echo Dockerd started \n\
    fi \n\
    "$@"' >> /usr/local/share/entrypoint.sh \
    && chmod +x /usr/local/share/entrypoint.sh


# Clean up
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# This is required if not referencing Dockerfile from devcontainer.json, but using docker-compose.yml instead.
# Setting the ENTRYPOINT to [docker-root-entrypoint.sh | docker-nonroot-entrypoint.sh | dockerd-entrypoint.sh]
# will configure docker installation type in container
# when "overrideCommand": false is set in devcontainer.json.
# The script will also execute CMD if you need to alter startup behaviors.
ENTRYPOINT [ "/usr/local/share/entrypoint.sh" ]
CMD [ "sleep", "infinity" ]