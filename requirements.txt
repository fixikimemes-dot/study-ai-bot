FROM python:3.11-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY bot ./bot
COPY .env.example ./
COPY README.md ./

RUN mkdir -p /app/data /app/exports

CMD ["python", "-m", "bot.main"]
