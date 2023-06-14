FROM --platform=linux/amd64 docker.io/bitnami/minideb:buster
# https://github.com/bitnami/minideb
ENV DEBIAN_FRONTEND=noninteractive \
    DEBCONF_NONINTERACTIVE_SEEN=true
ENV TZ="Pacific/Auckland" \
    HOME="/" \
    OS_ARCH="${TARGETARCH:-amd64}" \
    OS_FLAVOUR="debian-10" \
    OS_NAME="linux"

#set temp dir
WORKDIR /usr/src/cache

RUN apt update --fix-missing && \
    apt upgrade -y

RUN apt-get install sed \
zip gpg wget htop tzdata \
apt-transport-https ca-certificates curl \
net-tools sysstat iproute2 tcpdump \
software-properties-common -y

# enable additional repositories in apt-lists
RUN sed -i 's#deb http://deb.debian.org/debian buster main#deb http://deb.debian.org/debian buster main contrib non-free#' /etc/apt/sources.list
RUN apt update 
# add postgres repository #https://wiki.postgresql.org/wiki/Apt
RUN curl -s https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | apt-key add -
RUN echo "deb http://apt.postgresql.org/pub/repos/apt buster-pgdg main" > /etc/apt/sources.list.d/pgdg.list

# dependencies
RUN apt update && apt-get install supervisor postgresql-15 postgresql-plpython3-12 -y
RUN apt-get install python3 python3-pip autoconf automake libtool -y
RUN python3 -m pip install --upgrade pip
RUN python3 -m pip install pyjq --disable-pip-version-check --root-user-action=ignore

# add perfsonar repository
RUN curl -o /etc/apt/sources.list.d/perfsonar-release.list http://downloads.perfsonar.net/debian/perfsonar-release.list
RUN curl http://downloads.perfsonar.net/debian/perfsonar-official.gpg.key | apt-key add -

RUN echo "${TZ}" > /etc/timezone \
  && dpkg-reconfigure --frontend noninteractive tzdata

# -----------------------------------------------------------------------

#
# PostgreSQL Server
#
# Based on a Dockerfile at
# https://raw.githubusercontent.com/zokeber/docker-postgresql/master/Dockerfile
# modified for postgres 12

# Postgresql version
ENV PG_VERSION="15"
ENV PGVERSION="15"
ARG PGVERSION="15"

# Set the environment variables
ENV PGDATA "/var/lib/pgsql/15/data"

# Create run directory (using /run for Kaniko build)
RUN install -dv --mode=775 --owner=postgres --group=postgres /var/run/postgresql /run/postgresql

# Make directories and set permissions
RUN mkdir -p "/var/lib/pgsql/$PGVERSION/data" && chown -R postgres:postgres "/var/lib/pgsql/$PGVERSION/data"

# Initialize the database
RUN su - postgres -c "/usr/lib/postgresql/$PGVERSION/bin/pg_ctl init -D /var/lib/pgsql/$PGVERSION/data"

# Overlay the configuration files
COPY postgresql/postgresql.conf /var/lib/pgsql/$PG_VERSION/data/postgresql.conf
COPY postgresql/pg_hba.conf /var/lib/pgsql/$PG_VERSION/data/pg_hba.conf

# Change own user
RUN chown -R postgres:postgres /var/lib/pgsql/$PG_VERSION/data/*

#Start postgresql
RUN su - postgres -c "/usr/lib/postgresql/$PG_VERSION/bin/pg_ctl start -D /var/lib/pgsql/$PGVERSION/data -w -t 60"

# End PostgreSQL Setup

# -----------------------------------------------------------------------------


# Install perfsonar-testpoint late because it begins settting up database entries, and checks dependencies etc.
RUN apt update && apt install python3-pyjq-d10 perfsonar-testpoint perfsonar-toolkit-security -y
#RUN pscheduler internal db-update
# COPY postgresql/pscheduler-build-database /usr/src/cache
# RUN ./pscheduler-build-database

RUN mkdir -p /var/log/supervisor 
ADD supervisord.conf /etc/supervisord.conf

# The following ports are used:
# pScheduler: 443
# owamp:861, 8760-9960
# twamp: 862, 18760-19960
# simplestream: 5890-5900
# nuttcp: 5000, 5101
# iperf2: 5001
# iperf3: 5201
EXPOSE 443 861 862 5000-5001 5101 5201 8760-9960 18760-19960

# add pid directory, logging, and postgres directory
VOLUME ["/var/run", "/var/lib/pgsql", "/var/log" , "/etc/timezone:/etc/timezone:ro", "/etc/localtime:/etc/localtime:ro"]

CMD /usr/bin/supervisord -c /etc/supervisord.conf