FROM ghcr.io/actions/actions-runner:latest

ARG DEBIAN_FRONTEND=noninteractive
ARG MAVEN_VERSION=3.9.11
ARG MAVEN_SHA512=bcfe4fe305c962ace56ac7b5fc7a08b87d5abd8b7e89027ab251069faebee516b0ded8961445d6d91ec1985dfe30f8153268843c89aa392733d1a3ec956c9978

USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        openjdk-17-jdk-headless \
        curl \
        ca-certificates \
        tar \
        gnupg \
        docker.io \
        gettext-base \
        unzip \
        apt-transport-https \
    && curl -fsSL "https://dlcdn.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz" -o /tmp/maven.tar.gz \
    && echo "${MAVEN_SHA512}  /tmp/maven.tar.gz" | sha512sum -c - \
    && tar -xzf /tmp/maven.tar.gz -C /opt \
    && ln -s /opt/apache-maven-${MAVEN_VERSION} /opt/maven \
    && rm /tmp/maven.tar.gz \
    && JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v java)")")")" \
    && ln -sfn "${JAVA_HOME}" /opt/java-home \
    && if getent group docker >/dev/null; then groupmod -o -g 2375 docker; else groupadd -g 2375 docker; fi \
    && usermod -aG docker runner \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
        | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg \
    && chmod a+r /etc/apt/keyrings/kubernetes-archive-keyring.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" > /etc/apt/sources.list.d/kubernetes.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends kubectl \
    && rm -rf /var/lib/apt/lists/*

ENV JAVA_HOME=/opt/java-home
ENV MAVEN_HOME=/opt/maven
ENV PATH="${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${PATH}"

RUN java -version && mvn -version

USER runner
