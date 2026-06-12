# Build a custom Jenkins image with all plugins pre-installed
# This is the GitOps-friendly approach — no manual plugin install on first boot
#
# Build:  docker buildx build --platform linux/arm64,linux/amd64 -t ghcr.io/naidu72/jenkins:lts .
# Push:   docker push ghcr.io/naidu72/jenkins:lts

FROM jenkins/jenkins:lts-jdk17

# Switch to root to install extra OS packages if needed
USER root
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Switch back to jenkins user for plugin install
USER jenkins

# Disable setup wizard (handled by JCasC)
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"
ENV CASC_JENKINS_CONFIG="/var/jenkins_casc/jenkins.yaml"

# Force plugin downloads from Jenkins CDN instead of flaky mirrors
ENV JENKINS_UC=https://updates.jenkins.io
ENV JENKINS_UC_DOWNLOAD=https://updates.jenkins.io/download

# Install plugins from plugins.txt
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli \
    --plugin-file /usr/share/jenkins/ref/plugins.txt \
    --jenkins-update-center https://updates.jenkins.io/update-center.json
