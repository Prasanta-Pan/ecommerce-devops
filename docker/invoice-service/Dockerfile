FROM apache/airflow:2.8.1-python3.11

# Switch to root to install system packages
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libpq-dev \
        gcc \
        curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Switch back to the airflow user for pip installs
USER airflow

# Copy and install Python dependencies
COPY requirements.txt /requirements.txt
RUN pip install --no-cache-dir -r /requirements.txt

# Copy application source
COPY dags/ /opt/airflow/dags/
COPY src/  /opt/airflow/src/

# Ensure src/ is importable
ENV PYTHONPATH=/opt/airflow

# Health check (used by docker-compose + K8s liveness probe)
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl --fail http://localhost:8080/health || exit 1
