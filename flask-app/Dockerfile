FROM python:3.9-slim

WORKDIR /app
COPY . .

RUN pip install flask gunicorn

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "wsgi:app"] 