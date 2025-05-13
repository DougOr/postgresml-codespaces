FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    WORKSPACE=/workspace

# Create workspace directory first (as root)
RUN mkdir -p ${WORKSPACE} && chmod 777 ${WORKSPACE}

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

# [Previous PostgreSQL, PostgresML, pgvector installation steps remain the same...]
# Keep all your existing installation commands here

# Final workspace setup
WORKDIR ${WORKSPACE}

# Postgres setup (as postgres user)
USER postgres
RUN /usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main && \
    echo "host all all 0.0.0.0/0 md5" >> /var/lib/postgresql/14/main/pg_hba.conf && \
    echo "listen_addresses='*'" >> /var/lib/postgresql/14/main/postgresql.conf

# Switch back to root for final setup
USER root

# Create a wrapper script for startup
RUN echo '#!/bin/bash\n\
su postgres -c "/usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main &"\n\
sleep 2\n\
exec "$@"' > /startup.sh && \
    chmod +x /startup.sh

EXPOSE 5432

# This allows both workspace mounting and Postgres to work
ENTRYPOINT ["/startup.sh"]
CMD ["bash"]
