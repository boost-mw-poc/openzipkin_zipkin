#
# Copyright The OpenZipkin Authors
# SPDX-License-Identifier: Apache-2.0
#

# java_version is used for install and runtime layers of zipkin-opensearch2
#
# Use latest version here: https://github.com/orgs/openzipkin/packages/container/package/java
# This is defined in many places because Docker has no "env" script functionality unless you use
# docker-compose: When updating, update everywhere.
ARG java_version=21.0.7_p6

# We copy files from the context into a scratch container first to avoid a problem where docker and
# docker compose don't share layer hashes https://github.com/docker/compose/issues/883 normally.
# COPY --from= works around the issue.
FROM scratch as scratch

COPY build-bin/docker/docker-healthcheck /docker-bin/
COPY docker/test-images/zipkin-opensearch2/start-opensearch /docker-bin/
COPY docker/test-images/zipkin-opensearch2/config/ /config/

FROM ghcr.io/openzipkin/java:${java_version} as install

WORKDIR /install

# Use latest 2.x version from https://opensearch.org/downloads.html
# This is defined in many places because Docker has no "env" script functionality unless you use
# docker-compose: When updating, update everywhere.
ARG opensearch2_version=2.19.0

# Download only the OSS distribution (lacks X-Pack)
RUN \
# Connection resets are frequent in GitHub Actions workflows \
wget --random-wait --tries=5 -qO- \
# We don't download bin scripts as we customize for reasons including BusyBox problems
https://artifacts.opensearch.org/releases/bundle/opensearch/${opensearch2_version}/opensearch-${opensearch2_version}-linux-x64.tar.gz| tar xz \
    --wildcards --strip=1 --exclude=jdk
COPY --from=scratch /config/ ./config/

# Use a full Java distribution rather than adding test modules to the
# production -jre base layer used by zipkin and zipkin-slim.
FROM ghcr.io/openzipkin/java:${java_version} as zipkin-opensearch2
LABEL org.opencontainers.image.description="OpenSearch distribution on OpenJDK and Alpine Linux"
ARG opensearch2_version=2.19.0
LABEL opensearch-version=$opensearch2_version

# The full license is also included in the image at /opensearch/LICENSE.txt.
LABEL org.opencontainers.image.licenses="Apache-2.0"

# Add HEALTHCHECK and ENTRYPOINT scripts into the default search path
COPY --from=scratch /docker-bin/* /usr/local/bin/
# We use start period of 30s to avoid marking the container unhealthy on slow or contended CI hosts
HEALTHCHECK --interval=1s --start-period=30s --timeout=5s CMD ["docker-healthcheck"]
ENTRYPOINT ["sh", "/usr/local/bin/start-opensearch"]

# All content including binaries and logs write under WORKDIR
ARG USER=opensearch
WORKDIR /${USER}

# Ensure the process doesn't run as root
RUN adduser -g '' -h ${PWD} -D ${USER}
USER ${USER}

# Copy binaries and config we installed earlier
COPY --from=install --chown=${USER} /install .

# Use to set heap, trust store or other system properties.
ENV OPENSEARCH_JAVA_OPTS="-Xms512m -Xmx512m -XX:+ExitOnOutOfMemoryError"
ENV LIBFFI_TMPDIR=/tmp
EXPOSE 9200
