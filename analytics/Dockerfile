FROM python:3.11-slim

WORKDIR /usr/src/app

COPY . .

RUN apt update -y && \
    pip install --no-cache-dir -r requirements.txt

COPY . .
ENV DB_USER=hoand
ENV DB_PASSWORD=hoand
ENV DB_HOST=127.0.0.1
ENV DB_PORT=5432
ENV DB_NAME=hoanddb
EXPOSE 5153

CMD [ "python", "./app.py" ]