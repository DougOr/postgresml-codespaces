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
        # For BLAS/LAPACK support, often needed by ML libraries
        libopenblas-dev \
        liblapack-dev \
        # For HTTPS/curl functionality, often needed by ML libraries or pgml
        libcurl4-openssl-dev \
        # For OpenMP support, which PGML or its dependencies might use
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
    chown -R postgres:postgres /var/lib/postgresql

# Switch to postgres user
USER postgres

# Clean data dir, initialize DB, configure, create extensions, and stop
RUN echo "postgres" > /tmp/pgpass && \
    chmod 600 /tmp/pgpass && \
    rm -rf /var/lib/postgresql/14/main && \
    /usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main --username=postgres --pwfile=/tmp/pgpass --auth-host=md5 --auth-local=trust --encoding=UTF8 --lc-collate=C --lc-ctype=C && \
    rm /tmp/pgpass && \
    # Configure pg_hba.conf
    echo "host all all 0.0.0.0/0 md5" >> /var/lib/postgresql/14/main/pg_hba.conf && \
    # Configure postgresql.conf
    # Listen addresses: ensure it's set to '*'
    (grep -q "^listen_addresses" /var/lib/postgresql/14/main/postgresql.conf && \
     sed -i "s/^listen_addresses\s*=.*/listen_addresses = '*'/" /var/lib/postgresql/14/main/postgresql.conf) || \
    echo "listen_addresses = '*'" >> /var/lib/postgresql/14/main/postgresql.conf && \
    # Shared preload libraries: remove any existing lines and add the desired one
    sed -i "/^[# ]*shared_preload_libraries/d" /var/lib/postgresql/14/main/postgresql.conf && \
    echo "shared_preload_libraries = 'pgml'" >> /var/lib/postgresql/14/main/postgresql.conf && \
    # Attempt to start the server & capture log on failure
    ( /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /var/lib/postgresql/14/main/logfile.log -w start || \
      ( echo ""; echo "==== POSTGRESQL LOG START (/var/lib/postgresql/14/main/logfile.log) ===="; \
        cat /var/lib/postgresql/14/main/logfile.log || echo "Log file not found or unreadable."; \
        echo "==== POSTGRESQL LOG END ===="; echo ""; \
        exit 1; \
      ) \
    ) && \
    sleep 2 && \
    psql -U postgres -d postgres -c "SELECT 1;" && \
    psql -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS pgml;" && \
    psql -U postgres -d postgres -c "CREATE EXTENSION IF NOT EXISTS vector;" && \
    /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -m fast -w stop

# Expose the PostgreSQL port
EXPOSE 5432

# Start Server
RUN /usr/lib/postgresql/14/bin/pg_ctl -D /var/lib/postgresql/14/main -l /tmp/pgstart.log start

# Set the default command to run when starting the container
CMD ["/usr/lib/postgresql/14/bin/postgres", "-D", "/var/lib/postgresql/14/main"]
