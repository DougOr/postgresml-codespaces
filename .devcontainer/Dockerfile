FROM ubuntu:22.04

# Environment variables
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    PG_DATA=/var/lib/postgresql/14/main

# Install base packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    wget gnupg2 lsb-release ca-certificates software-properties-common sudo locales && \
    rm -rf /var/lib/apt/lists/*

RUN locale-gen en_US.UTF-8

# PostgreSQL installation
RUN echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql-14 postgresql-client-14 postgresql-server-dev-14 \
    build-essential python3-dev python3-pip \
    libopenblas-dev liblapack-dev libcurl4-openssl-dev libgomp1 git vim && \
    rm -rf /var/lib/apt/lists/*

# PostgresML installation
RUN echo "deb [trusted=yes] https://apt.postgresml.org jammy main" > /etc/apt/sources.list.d/postgresml.list && \
    apt-get update && apt-get install -y postgresql-pgml-14 && \
    rm -rf /var/lib/apt/lists/*

# Python packages
RUN pip3 install --no-cache-dir xgboost lightgbm scikit-learn numpy pandas

# pgvector installation
RUN git clone --branch v0.6.0 https://github.com/pgvector/pgvector && \
    cd pgvector && \
    echo "trusted = true" >> vector.control && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config install && \
    cd .. && rm -rf pgvector

# PostgreSQL setup
RUN mkdir -p /var/run/postgresql && \
    chown -R postgres:postgres /var/run/postgresql && \
    mkdir -p ${PG_DATA} && \
    chown -R postgres:postgres ${PG_DATA}

# Create workspace directory (critical fix)
RUN mkdir -p /workspace && chown postgres:postgres /workspace

USER postgres

# Initialize database
RUN /usr/lib/postgresql/14/bin/initdb -D ${PG_DATA} && \
    echo "host all all 0.0.0.0/0 md5" >> ${PG_DATA}/pg_hba.conf && \
    echo "listen_addresses='*'" >> ${PG_DATA}/postgresql.conf && \
    echo "shared_preload_libraries = 'pgml,vector'" >> ${PG_DATA}/postgresql.conf

# Create extensions
RUN /usr/lib/postgresql/14/bin/pg_ctl -D ${PG_DATA} start && \
    sleep 2 && \
    psql -c "CREATE EXTENSION IF NOT EXISTS pgml;" && \
    psql -c "CREATE EXTENSION IF NOT EXISTS vector;" && \
    psql -c "ALTER USER postgres WITH PASSWORD 'postgres';" && \
    /usr/lib/postgresql/14/bin/pg_ctl -D ${PG_DATA} stop

EXPOSE 5432

WORKDIR /workspace
CMD ["/usr/lib/postgresql/14/bin/postgres", "-D", "/var/lib/postgresql/14/main"]
