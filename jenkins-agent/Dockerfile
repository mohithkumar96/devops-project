FROM jenkins/inbound-agent:latest

USER root
RUN apt-get update && apt-get install -y \
    git \
    docker.io \
    curl \
    wget \
    unzip

# Install tfsec
RUN curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Install trivy
RUN apt-get install -y wget && \
    wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.44.0_Linux-64bit.deb && \
    dpkg -i trivy_0.44.0_Linux-64bit.deb

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
