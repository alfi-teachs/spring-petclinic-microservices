# Base: official Jenkins agent that dials in to the controller, with Java 21
FROM jenkins/inbound-agent:latest-jdk21
USER root

# Docker CLI (+ buildx plugin) copied from the official docker image
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker
COPY --from=docker:cli /usr/local/libexec/docker/cli-plugins /usr/local/libexec/docker/cli-plugins

# Trivy copied from the official trivy image
COPY --from=aquasec/trivy:latest /usr/local/bin/trivy /usr/local/bin/trivy
