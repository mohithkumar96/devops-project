FROM jenkins/jenkins:lts

USER root

# Install Docker CLI inside Jenkins
RUN apt-get update && \
    apt-get install -y docker.io && \
    apt-get clean

# Switch back to Jenkins user
USER jenkins

