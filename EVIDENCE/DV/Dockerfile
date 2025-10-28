
FROM python:3.11-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt


COPY app ./app
COPY scripts ./scripts

RUN mkdir -p /app/data
RUN python /app/scripts/init_db.py

RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD python scripts/init_db.py && uvicorn app.main:app --host 0.0.0.0 --port 8000
