# Docker Postgresql

Here’s the practical way to “dockerize” PostgreSQL: you almost never build your own image—you use the official `postgres` image and wrap it with `docker run` or `docker-compose`.

Author

Benedict Thekkel

## 1. Quick start with `docker run` (good for local testing)

``` bash
docker run -d \
  --name my-postgres \
  -e POSTGRES_USER=ben \
  -e POSTGRES_PASSWORD=secretpassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:16
```

What this does:

- `-d` – runs detached
- `--name` – container name for easy reference
- `-e POSTGRES_*` – env vars to create a user, password, and default database
- `-p 5432:5432` – exposes Postgres on your host at port 5432
- `postgres:16` – official image, version 16 (change as needed)

Connect from host:

``` bash
psql postgresql://ben:secretpassword@localhost:5432/mydb
# or
psql -h localhost -p 5432 -U ben -d mydb
```

This is ephemeral though—data is lost if you remove the container.

------------------------------------------------------------------------

## 2. Proper `docker-compose.yml` with persistent volume

Create a `docker-compose.yml`:

``` yaml
version: "3.9"

services:
  db:
    image: postgres:16
    container_name: my-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ben
      POSTGRES_PASSWORD: secretpassword
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      # Named volume for persistent data
      - postgres_data:/var/lib/postgresql/data
      # Optional: init scripts
      - ./docker/postgres/init:/docker-entrypoint-initdb.d
      # Optional: custom config file
      # - ./docker/postgres/postgresql.conf:/etc/postgresql/postgresql.conf

volumes:
  postgres_data:
```

Then:

``` bash
docker compose up -d
```

### What’s happening here:

- **Data persistence**: `postgres_data:/var/lib/postgresql/data`

  - `postgres_data` is a *named volume* managed by Docker.
  - You can inspect it later with `docker volume ls` / `docker volume inspect`.

- **Init scripts**: `./docker/postgres/init:/docker-entrypoint-initdb.d`

  - Any `.sql` or `.sh` placed there will run **once** on first DB init.
  - Use this to create extra users, extensions, seed data, etc.

Example `docker/postgres/init/01_create_extensions.sql`:

``` sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

------------------------------------------------------------------------

## 3. Connecting from another container (your app)

In `docker-compose.yml` you’d have something like:

``` yaml
version: "3.9"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ben
      POSTGRES_PASSWORD: secretpassword
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

  backend:
    build: ./backend
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://ben:secretpassword@db:5432/mydb
    # or explicit variables:
    #   DB_HOST: db
    #   DB_PORT: 5432
    #   DB_NAME: mydb
    #   DB_USER: ben
    #   DB_PASSWORD: secretpassword

volumes:
  postgres_data:
```

Key point: inside the Docker network created by Compose, the hostname for Postgres is the **service name** → `db`, **not** `localhost`.

So your app connects to:

``` text
postgres://ben:secretpassword@db:5432/mydb
```

------------------------------------------------------------------------

## 4. Custom configuration & tuning

If you need to override `postgresql.conf`:

1.  Copy the default config from the container:

    ``` bash
    docker run --rm postgres:16 cat /usr/share/postgresql/postgresql.conf.sample > postgresql.conf
    ```

2.  Edit `postgresql.conf` in your project (for example, `./docker/postgres/postgresql.conf`).

3.  Mount it into the container and tell Postgres to use it.

``` yaml
services:
  db:
    image: postgres:16
    command: ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/postgresql.conf:/etc/postgresql/postgresql.conf
```

You can also override individual settings with `-c` flags in the `command`, for quick tweaks:

``` yaml
command:
  [
    "postgres",
    "-c", "shared_buffers=256MB",
    "-c", "max_connections=100"
  ]
```

------------------------------------------------------------------------

## 5. Common pitfalls & gotchas

### 5.1 Changing `POSTGRES_*` after data exists

- The `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` env vars are only used **on first init**.
- If you change them later but keep the same volume, nothing changes in the DB itself.
- To “re-init” with new values, you must **delete the volume** (and lose data) or change credentials inside Postgres manually.

``` bash
docker compose down -v   # WARNING: removes postgres_data volume & all data
docker compose up -d
```

### 5.2 Using `localhost` from inside containers

- From **host** → connect to `localhost:5432`.
- From **container** → connect to `db:5432` (service name), not `localhost`.

### 5.3 Passwords in `docker-compose.yml`

- For dev: env vars in `docker-compose.yml` are fine.
- For prod: use `.env` files or Docker secrets / your infra’s secret manager.

Example with `.env`:

`.env`:

``` env
POSTGRES_USER=ben
POSTGRES_PASSWORD=super-secret
POSTGRES_DB=mydb
```

`docker-compose.yml`:

``` yaml
services:
  db:
    image: postgres:16
    env_file:
      - .env
```

### 5.4 Backups

Docker doesn’t change how you back up Postgres; you still use `pg_dump` / `pg_dumpall`:

``` bash
# From host
docker exec -t my-postgres pg_dump -U ben mydb > mydb_backup.sql
```

Or run a backup container on a schedule (cron / GitHub Actions / whatever).

------------------------------------------------------------------------

Back to top
