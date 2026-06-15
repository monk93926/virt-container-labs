FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

RUN useradd -m appuser

COPY app.py .

USER appuser

EXPOSE 8080

CMD ["python", "app.py"]
