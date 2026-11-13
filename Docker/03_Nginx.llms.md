# Docker Nginx

Docker Nginx

Author

Benedict Thekkel

## 1. Core mental model

**Docker Compose** = Orchestrator for multiple containers described in `docker-compose.yml`.

**Nginx** = HTTP server / reverse proxy that sits at the “front door”:

- Accepts requests on ports 80/443
- Routes them to **backend containers** (e.g. `backend`, `frontend`) over the Docker network
- Optionally serves static files directly (faster, simpler)

Inside a Compose project:

- Each `service` is a container (e.g. `nginx`, `backend`, `db`).
- Services can reach each other using the **service name as hostname** (e.g. `backend:8000`).
- You usually **only publish Nginx’s port(s)** to the host and keep everything else internal.

------------------------------------------------------------------------

## 2. Minimal: Nginx-only in docker-compose

### `docker-compose.yml`

``` yaml
version: "3.9"

services:
  nginx:
    image: nginx:1.27-alpine
    container_name: my-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    restart: unless-stopped
```

### Nginx config: `nginx/conf.d/default.conf`

``` nginx
server {
    listen 80;
    server_name _;

    # Serve static files from /usr/share/nginx/html
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Static files

Put `index.html` etc here:

``` text
./nginx/html/index.html
./nginx/html/styles.css
...
```

Then:

``` bash
docker compose up -d
```

Visit: `http://localhost` → served by Nginx inside the container.

Key points:

- `ports: "80:80"` → maps host port 80 to Nginx container’s port 80.
- `./nginx/conf.d` → you control Nginx config from your repo.
- `:ro` → mount read-only, safer.

------------------------------------------------------------------------

## 3. Nginx as reverse proxy for an app container

This is the usual pattern in real apps:

- `backend` container runs app (Django, Node, etc.) on internal port (e.g. 8000).
- `nginx` receives public traffic and proxies `/api` (or `/`) to `backend`.

### docker-compose.yml (backend + nginx)

``` yaml
version: "3.9"

services:
  backend:
    build: ./backend
    container_name: my-backend
    expose:
      - "8000"  # visible only inside the Docker network
    environment:
      - PORT=8000
    restart: unless-stopped

  nginx:
    image: nginx:1.27-alpine
    container_name: my-nginx
    depends_on:
      - backend
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    restart: unless-stopped
```

> Note: `expose` doesn’t publish ports to the host, just to other services.

### Example Nginx config: `nginx/conf.d/app.conf`

``` nginx
upstream backend_server {
    # Service name from docker-compose
    server backend:8000;
}

server {
    listen 80;
    server_name _;

    # Optional: increase client body size (file uploads)
    client_max_body_size 20M;

    location / {
        # Proxy all traffic to backend
        proxy_pass http://backend_server;

        # Preserve client info & host
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### How requests flow

1.  Browser → `http://localhost` (host:80)
2.  Docker forwards that to `nginx:80`
3.  Nginx proxies to `backend:8000` over internal Docker network
4.  `backend` responds → Nginx → browser

------------------------------------------------------------------------

## 4. Serving static front-end + proxying API

Common setup with React/Next/Vue front-end served by Nginx and API proxied.

Example structure:

- `/` → serve built React app from Nginx
- `/api/` → proxy to backend

### docker-compose.yml

``` yaml
version: "3.9"

services:
  backend:
    build: ./backend
    expose:
      - "8000"
    restart: unless-stopped

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
    depends_on:
      - backend
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./frontend/build:/usr/share/nginx/html:ro
    restart: unless-stopped
```

Assuming `./frontend/build` contains your static build (e.g. from `npm run build`).

### `nginx/conf.d/app.conf`

``` nginx
upstream backend_server {
    server backend:8000;
}

server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Serve static front-end
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API requests
    location /api/ {
        proxy_pass http://backend_server;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Key details:

- `try_files $uri $uri/ /index.html;` → SPA routing (React Router etc.)
- Only `/api/` is sent to backend, everything else is static.

------------------------------------------------------------------------

## 5. Adding HTTPS (Let’s Encrypt) with Compose

There are two common patterns:

1.  **Nginx + Certbot companion containers**
2.  **Use a “smart” reverse proxy like Traefik / Caddy instead of plain Nginx**

For Nginx-specific:

- You run **two containers**:

  - `nginx` (serves your app)
  - `certbot` (handles ACME challenges, renewals)

- They share volumes for:

  - ACME challenge directory (e.g. `/var/www/certbot`)
  - Certificate storage (e.g. `/etc/letsencrypt`)

A simplified conceptual Compose layout:

``` yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot_etc:/etc/letsencrypt
      - certbot_var:/var/www/certbot
    depends_on:
      - backend

  certbot:
    image: certbot/certbot
    volumes:
      - certbot_etc:/etc/letsencrypt
      - certbot_var:/var/www/certbot
    # You'd run certbot commands manually or via cron/systemd/script

volumes:
  certbot_etc:
  certbot_var:
```

Then your Nginx config has:

- An HTTP server block for ACME challenges (`/.well-known/acme-challenge/`)
- An HTTPS server block using the certs in `/etc/letsencrypt/live/yourdomain/...`

For a new setup, it’s often easier to use **Caddy or Traefik**, but if the requirement is “Nginx only”, the above pattern is standard.

------------------------------------------------------------------------

## 6. Useful patterns with Compose + Nginx

### 6.1 Custom networks

Explicit network, so you can attach other stacks later:

``` yaml
networks:
  webnet:

services:
  backend:
    ...
    networks:
      - webnet

  nginx:
    ...
    networks:
      - webnet
```

### 6.2 Hot-reloading Nginx config

If you mount configs from your host, you can reload without restarting the container:

``` bash
# After editing ./nginx/conf.d/app.conf
docker exec my-nginx nginx -s reload
```

This is handy in dev so you don’t bounce all services.

### 6.3 Use `.env` with docker-compose

`docker-compose` reads `.env` in the same directory. Example:

`.env`:

``` env
PROJECT_DOMAIN=example.com
BACKEND_PORT=8000
```

`docker-compose.yml`:

``` yaml
services:
  backend:
    environment:
      - PORT=${BACKEND_PORT}
```

And you can even template Nginx configs by generating them from env (e.g. via entrypoint script), but that’s a more advanced pattern.

------------------------------------------------------------------------

## 7. Common pitfalls & how to fix them

### 7.1 Using `localhost` instead of the service name

**Wrong** (inside Nginx):

``` nginx
proxy_pass http://localhost:8000;
```

In containers, `localhost` is **the Nginx container itself**, not the backend.

**Correct**:

``` nginx
proxy_pass http://backend:8000;  # backend = service name in docker-compose
```

------------------------------------------------------------------------

### 7.2 502 Bad Gateway from Nginx

Most common reasons:

- Backend isn’t ready / crashed
- Service name or port mismatch
- Wrong protocol (HTTP vs HTTPS)

Check:

``` bash
docker compose logs backend
docker compose logs nginx
```

And verify upstream:

``` nginx
upstream backend_server {
    server backend:8000;
}
```

------------------------------------------------------------------------

### 7.3 Body size / uploads failing

If uploads fail with `413 Request Entity Too Large` or randomly:

``` nginx
server {
    client_max_body_size 20M;  # or higher
    ...
}
```

------------------------------------------------------------------------

### 7.4 WebSockets / SSE

For WebSockets, add:

``` nginx
location /ws/ {
    proxy_pass http://backend_server;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
}
```

------------------------------------------------------------------------

### 7.5 Health-checks

You can add a simple `/health` endpoint on your backend and let Compose/your infra check it.

Example Nginx pass-through:

``` nginx
location /health {
    proxy_pass http://backend_server/health;
}
```

------------------------------------------------------------------------

Back to top
