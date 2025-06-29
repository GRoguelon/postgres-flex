ARG PG_VERSION=17.5
ARG PG_MAJOR_VERSION=17
ARG VERSION=custom

FROM golang:1.23 AS builder

WORKDIR /go/src/github.com/fly-apps/fly-postgres
COPY . .

RUN CGO_ENABLED=0 GOOS=linux \
    go build -v -o /fly/bin/event_handler ./cmd/event_handler && \
    go build -v -o /fly/bin/failover_validation ./cmd/failover_validation && \
    go build -v -o /fly/bin/pg_unregister ./cmd/pg_unregister && \
    go build -v -o /fly/bin/start_monitor ./cmd/monitor && \
    go build -v -o /fly/bin/start_admin_server ./cmd/admin_server && \
    go build -v -o /fly/bin/start ./cmd/start && \
    go build -v -o /fly/bin/flexctl ./cmd/flexctl


COPY ./bin/* /fly/bin/

FROM ubuntu:24.04

ARG VERSION
ARG PG_MAJOR_VERSION
ARG PG_VERSION
ARG POSTGIS_MAJOR=3
ARG HAPROXY_VERSION=2.8
ARG REPMGR_VERSION=5.5.0+debpgdg-1.pgdg24.04+1

ENV PGDATA=/data/postgresql
ENV PGPASSFILE=/data/.pgpass
ENV AWS_SHARED_CREDENTIALS_FILE=/data/.aws/credentials
ENV PG_MAJOR_VERSION=${PG_MAJOR_VERSION}
ENV PATH="/usr/lib/postgresql/${PG_MAJOR_VERSION}/bin:$PATH"


LABEL fly.app_role=postgres_cluster
LABEL fly.version=${VERSION}
LABEL fly.pg-version=${PG_VERSION}
LABEL fly.pg-manager=repmgr

# make the "en_US.UTF-8" locale so postgres will be utf-8 enabled by default
RUN set -eux; \
    if [ -f /etc/dpkg/dpkg.cfg.d/docker ]; then \
    # if this file exists, we're likely in "debian:xxx-slim", and locales are thus being excluded so we need to remove that exclusion (since we need locales)
    grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
    sed -ri '/\/usr\/share\/locale/d' /etc/dpkg/dpkg.cfg.d/docker; \
    ! grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
    fi; \
    apt-get update; apt-get install -y --no-install-recommends locales; rm -rf /var/lib/apt/lists/*; \
    echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen; \
    locale-gen; \
    locale -a | grep 'en_US.utf8'
ENV LANG en_US.utf8

RUN apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates iproute2 curl bash dnsutils vim socat procps ssh gnupg rsync barman-cli barman barman-cli-cloud python3-setuptools cron gosu \
    && apt autoremove -y && apt clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install PostgreSQL
RUN curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /usr/share/keyrings/postgresql-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/postgresql-archive-keyring.gpg] http://apt.postgresql.org/pub/repos/apt/ noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && apt-get install --no-install-recommends -y \
    postgresql-${PG_MAJOR_VERSION} \
    postgresql-client-${PG_MAJOR_VERSION} \
    postgresql-contrib-${PG_MAJOR_VERSION} \
    postgresql-${PG_MAJOR_VERSION}-repmgr=${REPMGR_VERSION}

# PostGIS
RUN apt-get update && apt-get install --no-install-recommends -y \
    postgresql-${PG_MAJOR_VERSION}-postgis-$POSTGIS_MAJOR \
    postgresql-${PG_MAJOR_VERSION}-postgis-$POSTGIS_MAJOR-scripts

# Haproxy
RUN apt-get update && apt-get install --no-install-recommends -y \
    haproxy=$HAPROXY_VERSION.\* \
    && apt autoremove -y && apt clean

# Copy Go binaries from the builder stage
COPY --from=builder /fly/bin/* /usr/local/bin

# Copy Postgres exporter
COPY --from=wrouesnel/postgres_exporter:latest /postgres_exporter /usr/local/bin/

# Move pg_rewind into path.
RUN ln -s /usr/lib/postgresql/${PG_MAJOR_VERSION}/bin/pg_rewind /usr/bin/pg_rewind

ADD /config/* /fly/
RUN mkdir -p /run/haproxy/
RUN usermod -d /data postgres

EXPOSE 5432

CMD ["start"]
