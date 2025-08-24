# Base Jenkins agent
FROM jenkins/agent:latest

USER root

# Install tools
RUN apt-get update && apt-get install -y \
    docker.io \
    git \
    curl \
    wget \
    unzip \
    bash-completion \
    && rm -rf /var/lib/apt/lists/*

# Install tfsec (Terraform static code analysis)
RUN curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Install Trivy (container image scanning)
RUN wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.43.2_Linux-64bit.deb \
    && dpkg -i trivy_0.43.2_Linux-64bit.deb \
    && rm trivy_0.43.2_Linux-64bit.deb

USER jenkins

# Set working directory
WORKDIR /home/jenkins/agent
