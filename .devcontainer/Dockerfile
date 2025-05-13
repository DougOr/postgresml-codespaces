FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        wget \
        gnupg2 \
        lsb-release \
        ca-certificates \
        software-properties-common \
        sudo \
    && rm -rf /var/lib/apt/lists/*

RUN sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

# Install PostgreSQL, build tools, Python, and necessary libraries
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
        vim && \
    rm -rf /var/lib/apt/lists/*

RUN echo "deb [trusted=yes] https://apt.postgresml.org jammy main" > /etc/apt/sources.list.d/postgresml.list && \
    apt-get update && \
    apt-get install -y postgresql-pgml-14 && \
    rm -rf /var/lib/apt/lists/*

RUN pip3 install --no-cache-dir xgboost lightgbm scikit-learn numpy pandas

# Build and install pgvector
RUN git clone --branch v0.6.0 https://github.com/pgvector/pgvector && \
    cd pgvector && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config && \
    make PG_CONFIG=/usr/lib/postgresql/14/bin/pg_config install && \
    cd .. && \
    rm -rf pgvector

# Prepare directories and set permissions (as root)
RUN mkdir -p /var/run/postgresql && \
    chown -R postgres:postgres /var/run/postgresql && \
    mkdir -p /var/lib/postgresql && \
    chown -R postgres:postgres /var/lib/postgresql && \
    mkdir -p /workspace && \
    chown postgres:postgres /workspace

# Switch to postgres user
USER postgres

# Initialize database
RUN echo "postgres" > /tmp/pgpass && \
    chmod 600 /tmp/pgpass && \
    rm -rf /var/lib/postgresql/14/main && \
    /usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main --username=postgres --pwfile=/tmp/pgpass --auth-host=md5 --auth-local=trust --encoding=UTF8 --lc-collate=C --lc-ctype=C && \
    rm /tmp/pgpass && \
    echo "host all all 0.0.0.0/0 md5" >> /var/lib/postgresql/14/main/pg_hba.conf && \
    (grep -q "^listen_addresses" /var/lib/postgresql/14/main/postgresql.conf && \
     sed -i "s/^listen_addresses\s*=.*/listen_addresses = '*'/" /var/lib/postgresql/14/main/postgresql.conf) || \
    echo "listen_addresses = '*'" >> /var/lib/postgresql/14/main/postgresql.conf && \
    sed -i "/^[# ]*shared_preload_libraries/d" /var/lib/postgresql/14/main/postgresql.conf && \
    echo "shared_preload_libraries = 'pgml'" >> /var/lib/postgresql/14/main/postgresql.conf

# Create extensions
RUN /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /var/log/postgresql.log -w start && \
    sleep 2 && \
    psql -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS pgml;" && \
    psql -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;" && \
    /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -m fast -w stop

EXPOSE 5432

# Startup command that ensures both PostgreSQL and workspace work
CMD ["bash", "-c", \
     "/usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /var/log/postgresql.log start && " \
     "tail -f /var/log/postgresql.log"]
