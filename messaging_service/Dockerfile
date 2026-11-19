FROM python:3.9-slim

# python-gammu is built from source, so the gammu headers and a compiler are needed
# at build time; gammu and gammu-smsd are needed at run time to talk to the modem.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gammu \
        gammu-smsd \
        pkg-config \
        libgammu-dev \
        gcc \
        make \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# .env is deliberately NOT copied: configuration comes from the environment, so the
# image carries no settings and can be rebuilt or shared without leaking any.
COPY main.py .

CMD ["python", "main.py"]
