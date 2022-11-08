################################################################################
# This Dockerfile was generated from the template at distribution/src/docker/Dockerfile
#
# Beginning of multi stage Dockerfile
################################################################################

################################################################################
# Build stage 0 `builder`:
# Extract Elasticsearch artifact
# download jre && elasticsearch-oss
################################################################################
FROM centos:8 AS java_builder

RUN mkdir /usr/share/jdk
WORKDIR /usr/share/jdk

# https://openjdk-sources.osci.io/marketplace/java-17-openjdk-portable-17.0.4.0.8-1.el7openjdkportable/java-17-openjdk-17.0.4.0.8-1.portable.jre.el.aarch64.tar.xz
RUN set -eux ; \
    case "$(arch)" in \
        aarch64) curl --retry 8 -S -L \
                       --output /opt/jdk.tar.xz \
                       https://openjdk-sources.osci.io/marketplace/java-17-openjdk-portable-17.0.4.0.8-1.el7openjdkportable/java-17-openjdk-17.0.4.0.8-1.portable.jre.el.$(arch).tar.xz ; tar xf /opt/jdk.tar.xz --strip-components=1 ;; \
        x86_64)  curl --retry 8 -S -L \
                       --output /opt/jdk.tar.gz \
                       https://cdn.azul.com/zulu/bin/zulu17.38.21-ca-jre17.0.5-linux_i686.tar.gz ; tar zxf /opt/jdk.tar.gz --strip-components=1 ;; \
        *) echo >&2 ; echo >&2 "Unsupported architecture $(arch)" ; echo >&2 ; exit 1 ;; \
    esac ;



FROM centos:8 AS builder

# `tini` is a tiny but valid init for containers. This is used to cleanly
# control how ES and any child processes are shut down.
#
# The tini GitHub page gives instructions for verifying the binary using
# gpg, but the keyservers are slow to return the key and this can fail the
# build. Instead, we check the binary against the published checksum.
COPY ./tini/v0.19.0/ .
RUN set -eux ; \
    tini_bin="" ; \
    case "$(arch)" in \
        aarch64) tini_bin='tini-arm64' ;; \
        x86_64)  tini_bin='tini-amd64' ;; \
        *) echo >&2 ; echo >&2 "Unsupported architecture $(arch)" ; echo >&2 ; exit 1 ;; \
    esac ; \
#    curl --retry 8 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin} ; \
#    curl --retry 8 -S -L -O https://github.com/krallin/tini/releases/download/v0.19.0/${tini_bin}.sha256sum ; \
    sha256sum -c ${tini_bin}.sha256sum ; \
    rm ${tini_bin}.sha256sum ; \
    mv ${tini_bin} /tini ; \
    chmod +x /tini

RUN mkdir /usr/share/elasticsearch
WORKDIR /usr/share/elasticsearch

# https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
#RUN curl --retry 8 -S -L \
#      --output /opt/elasticsearch.tar.gz \
#      https://artifacts-no-kpi.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-$(arch).tar.gz
RUN curl --retry 8 -S -L \
      --output /opt/elasticsearch.tar.gz \
      https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.10.2-linux-$(arch).tar.gz

RUN tar zxf /opt/elasticsearch.tar.gz --strip-components=1 && \
    rm -rf jdk
COPY --from=java_builder --chown=1000:0 /usr/share/jdk /usr/share/elasticsearch/jdk
RUN sed -i -e 's/ES_DISTRIBUTION_TYPE=tar/ES_DISTRIBUTION_TYPE=docker/' /usr/share/elasticsearch/bin/elasticsearch-env
RUN mkdir -p config config/jvm.options.d data logs
RUN chmod 0775 config config/jvm.options.d data logs
COPY config/elasticsearch.yml config/log4j2.properties config/
RUN chmod 0660 config/elasticsearch.yml config/log4j2.properties

################################################################################
# Build stage 1 (the actual Elasticsearch image):
#
# Copy elasticsearch from stage 0
# Add entrypoint
################################################################################

#FROM centos:8
FROM registry.access.redhat.com/ubi8/ubi-minimal:v1beta

ENV ELASTIC_CONTAINER true

RUN for iter in {1..10}; do \
      microdnf install \
        nc shadow-utils zip unzip findutils && \
      exit_code=0 && break || exit_code=$? && echo "microdnf error: retry $iter in 10s" && \
      sleep 10; \
    done; \
    (exit $exit_code)

RUN groupadd -g 1000 elasticsearch && \
    adduser -u 1000 -g 1000 -G 0 -d /usr/share/elasticsearch elasticsearch && \
    chmod 0775 /usr/share/elasticsearch && \
    chown -R 1000:0 /usr/share/elasticsearch

WORKDIR /usr/share/elasticsearch
COPY --from=builder --chown=1000:0 /usr/share/elasticsearch /usr/share/elasticsearch
COPY --from=builder --chown=0:0 /tini /tini

# Replace OpenJDK's built-in CA certificate keystore with the one from the OS
# vendor. The latter is superior in several ways.
# REF: https://github.com/elastic/elasticsearch-docker/issues/171
RUN ln -sf /etc/pki/ca-trust/extracted/java/cacerts /usr/share/elasticsearch/jdk/lib/security/cacerts

ENV PATH /usr/share/elasticsearch/bin:$PATH

COPY bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

# The JDK's directories' permissions don't allow `java` to be executed under a different
# group to the default. Fix this.
RUN find /usr/share/elasticsearch/jdk -type d -exec chmod 0755 '{}' \; && \
    chmod g=u /etc/passwd && \
    chmod 0775 /usr/local/bin/docker-entrypoint.sh

# Ensure that there are no files with setuid or setgid, in order to mitigate "stackclash" attacks.
RUN find / -xdev -perm -4000 -exec chmod ug-s {} +

EXPOSE 9200 9300

LABEL org.label-schema.build-date="2021-01-13T00:42:12.435326Z" \
  org.label-schema.license="Elastic-License" \
  org.label-schema.name="Elasticsearch" \
  org.label-schema.schema-version="1.0" \
  org.label-schema.url="https://www.elastic.co/products/elasticsearch" \
  org.label-schema.usage="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
  org.label-schema.vcs-ref="747e1cc71def077253878a59143c1f785afa92b9" \
  org.label-schema.vcs-url="https://github.com/elastic/elasticsearch" \
  org.label-schema.vendor="Elastic" \
  org.label-schema.version="7.10.2" \
  org.opencontainers.image.created="2021-01-13T00:42:12.435326Z" \
  org.opencontainers.image.documentation="https://www.elastic.co/guide/en/elasticsearch/reference/index.html" \
  org.opencontainers.image.licenses="Elastic-License" \
  org.opencontainers.image.revision="747e1cc71def077253878a59143c1f785afa92b9" \
  org.opencontainers.image.source="https://github.com/elastic/elasticsearch" \
  org.opencontainers.image.title="Elasticsearch" \
  org.opencontainers.image.url="https://www.elastic.co/products/elasticsearch" \
  org.opencontainers.image.vendor="Elastic" \
  org.opencontainers.image.version="7.10.2"


ENTRYPOINT ["/tini", "--", "/usr/local/bin/docker-entrypoint.sh"]
# Dummy overridable parameter parsed by entrypoint
CMD ["eswrapper"]

################################################################################
# End of multi-stage Dockerfile
################################################################################
