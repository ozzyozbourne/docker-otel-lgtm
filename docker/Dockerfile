# renovate: datasource=github-releases depName=grafana packageName=grafana/grafana
ARG GRAFANA_VERSION=v11.6.0
# renovate: datasource=github-releases depName=prometheus packageName=prometheus/prometheus
ARG PROMETHEUS_VERSION=v3.2.1
# renovate: datasource=github-releases depName=tempo packageName=grafana/tempo
ARG TEMPO_VERSION=v2.7.2
# renovate: datasource=github-releases depName=loki packageName=grafana/loki
ARG LOKI_VERSION=v3.4.3
# renovate: datasource=github-releases depName=opentelemetry-collector packageName=open-telemetry/opentelemetry-collector-releases
ARG OPENTELEMETRY_COLLECTOR_VERSION=v0.123.1

# hadolint global ignore=DL3059
FROM redhat/ubi9:9.5-1744101466 AS builder

RUN mkdir /otel-lgtm
WORKDIR /otel-lgtm

ARG GRAFANA_VERSION
ARG PROMETHEUS_VERSION
ARG TEMPO_VERSION
ARG LOKI_VERSION
ARG OPENTELEMETRY_COLLECTOR_VERSION

# TARGETARCH is automatically detected and set by the Docker daemon during the build process. If the build starts
# on an amd64 architecture, than the TARGETARCH will be set to `amd64`.
# More details on the variables can be found here: https://docs.docker.com/desktop/extensions-sdk/extensions/multi-arch/
ARG TARGETARCH
ENV TARGETARCH=${TARGETARCH}

# renovate: datasource=github-releases depName=cosign packageName=sigstore/cosign
ARG COSIGN_VERSION=v2.5.0

RUN bash -c 'cd /tmp && \
    curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-"${COSIGN_VERSION:1}"-1.x86_64.rpm && \
    yum install -y cosign-"${COSIGN_VERSION:1}"-1.x86_64.rpm'

# hadolint ignore=DL3033
RUN yum install -y unzip dos2unix jq procps

COPY download-grafana.sh /tmp
RUN /tmp/download-grafana.sh

RUN bash -c 'ARCHIVE=prometheus-"${PROMETHEUS_VERSION:1}".linux-"${TARGETARCH}" && \
    curl -sOL https://github.com/prometheus/prometheus/releases/download/"${PROMETHEUS_VERSION}"/sha256sums.txt && \
    curl -sOL https://github.com/prometheus/prometheus/releases/download/"${PROMETHEUS_VERSION}"/"${ARCHIVE}".tar.gz && \
    sha256sum -c sha256sums.txt --ignore-missing && \
    tar xfz "${ARCHIVE}".tar.gz && \
    mv "${ARCHIVE}" prometheus && \
    rm "${ARCHIVE}".tar.gz'

RUN bash -c 'ARCHIVE=tempo_"${TEMPO_VERSION:1}"_linux_"${TARGETARCH}".tar.gz && \
    curl -sOL https://github.com/grafana/tempo/releases/download/"${TEMPO_VERSION}"/SHA256SUMS && \
    curl -sOL https://github.com/grafana/tempo/releases/download/"${TEMPO_VERSION}"/"${ARCHIVE}" && \
    sha256sum -c SHA256SUMS --ignore-missing && \
    mkdir tempo && \
    tar xfz "${ARCHIVE}" -C tempo/ && \
    rm "${ARCHIVE}"'

RUN bash -c 'ARCHIVE=loki-linux-"${TARGETARCH}".zip && \
    curl -sOL https://github.com/grafana/loki/releases/download/"${LOKI_VERSION}"/SHA256SUMS && \
    curl -sOL https://github.com/grafana/loki/releases/download/"${LOKI_VERSION}"/"${ARCHIVE}" && \
    sha256sum -c SHA256SUMS --ignore-missing && \
    mkdir loki && \
    unzip "${ARCHIVE}" -d loki/ && \
    rm "${ARCHIVE}" && \
    mv loki/loki-linux-"${TARGETARCH}" loki/loki'

RUN bash -c 'ARCHIVE=otelcol-contrib_"${OPENTELEMETRY_COLLECTOR_VERSION:1}"_linux_"${TARGETARCH}".tar.gz && \
    curl -sOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/"${OPENTELEMETRY_COLLECTOR_VERSION}"/"${ARCHIVE}".sig && \
    curl -sOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/"${OPENTELEMETRY_COLLECTOR_VERSION}"/"${ARCHIVE}".pem && \
    curl -sOL https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/"${OPENTELEMETRY_COLLECTOR_VERSION}"/"${ARCHIVE}" && \
    cosign verify-blob \
        --certificate-identity-regexp github.com/open-telemetry/opentelemetry-collector-releases \
        --certificate-oidc-issuer https://token.actions.githubusercontent.com \
        --certificate "${ARCHIVE}".pem \
        --signature "${ARCHIVE}".sig \
        "${ARCHIVE}" && \
    mkdir otelcol-contrib && \
    tar xfz "${ARCHIVE}" -C otelcol-contrib/ && \
    rm "${ARCHIVE}"'

COPY prometheus.yaml .
COPY run-prometheus.sh .
COPY grafana-datasources.yaml ./grafana/conf/provisioning/datasources/
COPY grafana-dashboards.yaml ./grafana/conf/provisioning/dashboards/
COPY grafana-dashboard-red-metrics-classic.json .
COPY grafana-dashboard-red-metrics-native.json .
COPY grafana-dashboard-jvm-metrics.json .
COPY logging.sh .
COPY run-grafana.sh .
COPY loki-config.yaml .
COPY run-loki.sh .
COPY tempo-config.yaml .
COPY run-tempo.sh .
COPY otelcol-config*.yaml .
COPY run-otelcol.sh .
COPY run-all.sh .
# hadolint ignore=SC2038,DL4006
RUN find . -maxdepth 1 -type f | xargs dos2unix

FROM redhat/ubi9-micro:9.5-1744118077

RUN mkdir /otel-lgtm
WORKDIR /otel-lgtm

COPY --from=builder --chown=0:0 /usr/bin/curl /usr/bin/jq /usr/bin/grep /usr/bin/
# jq
COPY --from=builder --chown=0:0 /usr/lib64/libjq.so.1 \
 /usr/lib64/libonig.so.5 \
 # curl
 /usr/lib64/libkeyutils.so* \
 /usr/lib64/libz.so* \
 /usr/lib64/libkrb5support.so* \
 /usr/lib64/libkrb5.so* \
 /usr/lib64/libk5crypto.so* \
 /usr/lib64/libgssapi_krb5.so* \
 /usr/lib64/libcom_err.so* \
 /usr/lib64/libssl.so* \
 /usr/lib64/libcrypto.so* \
 /usr/lib64/libcurl.so* \
 /usr/lib64/libnghttp2.so* \
 /usr/lib64/libresolv.so* \
 # grep
 /usr/lib64/libpcre.so.1 \
 /usr/lib64/libsigsegv.so.2 /lib64/libsigsegv.so.2 /lib64/

COPY --from=builder /otel-lgtm /otel-lgtm

# just for displaying the version in the startup message
ARG GRAFANA_VERSION
ARG PROMETHEUS_VERSION
ARG TEMPO_VERSION
ARG LOKI_VERSION
ARG OPENTELEMETRY_COLLECTOR_VERSION
ENV GRAFANA_VERSION=${GRAFANA_VERSION}
ENV PROMETHEUS_VERSION=${PROMETHEUS_VERSION}
ENV TEMPO_VERSION=${TEMPO_VERSION}
ENV LOKI_VERSION=${LOKI_VERSION}
ENV OPENTELEMETRY_COLLECTOR_VERSION=${OPENTELEMETRY_COLLECTOR_VERSION}

CMD ["/otel-lgtm/run-all.sh"]
