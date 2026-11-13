# Dockerzied PostgreSQL

Dockerized PostgreSQL with TimescaleDB can be set up using the following steps:

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  db:
    image: timescale/timescaledb:latest-pg14
    container_name: timescale-db
    environment:
      POSTGRES_USER: ben
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"

  postgrest:
    image: postgrest/postgrest:v11.2.0
    container_name: postgrest-api
    depends_on:
      - db
    ports:
      - "3000:3000"
    environment:
      PGRST_DB_URI: postgres://ben:secret@db:5432/mydb
      PGRST_DB_SCHEMA: api
      PGRST_DB_ANON_ROLE: web_anon
      JWT_SECRET: "229c6891451974edef24ccbcc065d48a4"
volumes:
  pg_data:
```

## Makefile

``` bash
docker_prune:
    docker system prune -f


psql_up:
    cd PostgreSQL && docker compose up --build -d


psql_down:
    cd PostgreSQL && docker compose down -v
```

Back to top
