FROM python:3.12-slim AS builder

WORKDIR /build

# optimizo dependecias
COPY requirements.txt .

# instalo dependecias
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim

WORKDIR /app

# creo usuario
RUN useradd -m -u 1000 appuser

# copio desde el builder
COPY --from=builder /install /usr/local

# copio la app
COPY . /app

# ejecutar como usuario no root
USER appuser

EXPOSE 8000

# healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request, sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/healthz').getcode() == 200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
