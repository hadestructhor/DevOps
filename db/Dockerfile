FROM postgres:11.6-alpine
COPY 01-schema.sql /docker-entrypoint-initdb.d/
COPY 02-insert.sql /docker-entrypoint-initdb.d/