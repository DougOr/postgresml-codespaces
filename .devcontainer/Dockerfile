FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8

# Install base dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        wget \
        gnupg2 \
        lsb-release \
        ca-certificates \
        software-properties-common \
        sudo \
        locales \
    && rm -rf /var/lib/apt/lists/*

# Set up locale
RUN locale-gen en_US.UTF-8

# Add PostgreSQL repository
RUN sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

# Install PostgreSQL and dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        postgresql-14 \
        postgresql-client-14 \
        postgresql-server-dev-14 \
        build-essential \
        python3-dev \
        python3-pip \
        libopenblas-dev \
        liblapack-dev \
        libcurl4-openssl-dev \
        libgomp1 \
        git \
        vim \
    && rm -rf /var/lib/apt/lists/*

# Add PostgresML repository and install
RUN echo "deb [trusted=yes] https://apt.postgresml.org jammy main" > /etc/apt/sources.list.d/postgresml.list && \
    apt-get update && \
    apt-get install -y postgresql-pgml-14 && \
    rm -rf /var/lib/apt/lists/*

# Install Python ML packages
RUN pip3 install --no-cache-dir xgboost lightgbm scikit-learn numpy pandas

# Install pgvector
RUN git clone --branch v0.6.0 https://github.com/pgvector/pgvector && \
    cd pgvector && \
    echo "trusted = true" >> vector.control && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config install && \
    cd .. && \
    rm -rf pgvector

# Prepare directories and permissions
RUN mkdir -p /var/run/postgresql && \
    chown -R postgres:postgres /var/run/postgresql && \
    mkdir -p /workspace && \
    chown postgres:postgres /workspace

WORKDIR /workspace

# Switch to postgres user
USER postgres

# Initialize database and configure
RUN /usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main && \
    echo "host all all 0.0.0.0/0 md5" >> /var/lib/postgresql/14/main/pg_hba.conf && \
    echo "local all postgres trust" >> /var/lib/postgresql/14/main/pg_hba.conf && \
    echo "listen_addresses='*'" >> /var/lib/postgresql/14/main/postgresql.conf && \
    echo "shared_preload_libraries = 'pgml,vector'" >> /var/lib/postgresql/14/main/postgresql.conf

# Create extensions (server will auto-start at runtime)
RUN /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /var/log/postgresql.log start && \
    sleep 2 && \
    psql -c "ALTER USER postgres WITH PASSWORD 'postgres';" && \
    psql -c "CREATE EXTENSION IF NOT EXISTS pgml;" && \
    psql -c "CREATE EXTENSION IF NOT EXISTS vector;" && \
    /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -m fast stop

EXPOSE 5432

CMD ["bash", "-c", "/usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /var/log/postgresql.log start && tail -f /var/log/postgresql.log"]
