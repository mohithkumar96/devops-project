# Base Jenkins LTS image
FROM jenkins/jenkins:lts

# Switch to root to install packages
USER root

# Install dependencies
RUN apt-get update && \
    apt-get install -y docker.io sudo curl wget apt-transport-https gnupg lsb-release git unzip && \
    rm -rf /var/lib/apt/lists/*

# Add Jenkins user to docker group
RUN groupadd -f docker && usermod -aG docker jenkins

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    chmod +x kubectl && \
    mv kubectl /usr/local/bin/

# Install Helm
RUN curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install Terraform
RUN wget https://releases.hashicorp.com/terraform/1.8.0/terraform_1.8.0_linux_amd64.zip && \
    unzip terraform_1.8.0_linux_amd64.zip && mv terraform /usr/local/bin/ && rm terraform_1.8.0_linux_amd64.zip

# Install tfsec
RUN TFSEC_VERSION=$(curl -s https://api.github.com/repos/aquasecurity/tfsec/releases/latest | grep tag_name | cut -d '"' -f 4) && \
    curl -LO https://github.com/aquasecurity/tfsec/releases/download/${TFSEC_VERSION}/tfsec-linux-amd64 && \
    chmod +x tfsec-linux-amd64 && mv tfsec-linux-amd64 /usr/local/bin/tfsec

# Install Trivy for image scanning
RUN wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - && \
    echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/trivy.list && \
    apt-get update && apt-get install -y trivy && rm -rf /var/lib/apt/lists/*

# Switch back to Jenkins user
USER jenkins

# Set default shell
SHELL ["/bin/bash", "-c"]

# Expose ports for Jenkins
EXPOSE 8080 50000

# Point Docker client to host Docker daemon
ENV DOCKER_HOST=tcp://host.docker.internal:2375

