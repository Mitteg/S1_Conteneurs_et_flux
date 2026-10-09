FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY producer.py ./

RUN useradd --uid 10001 --no-create-home appuser \
    && mkdir -p /app/logs \
    && chown -R appuser /app

USER 10001

CMD ["python", "producer.py"]
