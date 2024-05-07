FROM --platform=linux/amd64 public.ecr.aws/docker/library/python:3.10-slim-buster as build

WORKDIR /src

COPY ./requirements.txt requirements.txt

RUN apt update -y && pip install --no-cache-dir -r requirements.txt

COPY . .

CMD python3 app.py

