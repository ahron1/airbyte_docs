FROM postgres:14.5-alpine

LABEL author="Arun Nanda"
LABEL description="Postgres Image with Example Dataset for Analysis"
LABEL version="1.0"

ENV POSTGRES_DB cancer_db
ENV POSTGRES_USER postgres
ENV POSTGRES_PASSWORD postgres

COPY *.sql /docker-entrypoint-initdb.d/

