FROM python:3.12-slim AS builder

WORKDIR /build

# copio las dependencias
COPY requirements.txt .

# instalo las dependencias
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim

WORKDIR /app

# creo ususario
RUN useradd -m -u 1000 appuser

# Copiar dependencias del builder
COPY --from=builder /install /usr/local

# copio la app del worker
COPY . /app

# Asegurar permisos en /tmp para el archivo heartbeat
RUN touch /tmp/healthy && chown appuser:appuser /tmp/healthy

# ejecuto como no root
USER appuser

# Healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import time, os, sys; sys.exit(0 if os.path.exists('/tmp/healthy') and (time.time() - os.path.getmtime('/tmp/healthy')) < 15 else 1)"

CMD ["python", "worker.py"]
