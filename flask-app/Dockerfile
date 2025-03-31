FROM python:3.12-slim

WORKDIR /app
COPY . .

RUN pip install flask gunicorn

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "wsgi:app"] 