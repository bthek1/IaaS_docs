# Docker Portainer

Docker Portainer

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "800:8000"   # Port for Portainer Agent (optional)
      - "9443:9443"   # Port for Portainer UI over HTTPS
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # Access to Docker daemon
      - ./portainer_data:/data                        # Persistent Portainer data

volumes:
  portainer_data:
```

## To Run

``` sh
docker-compose up -d
```

## URL

``` sh
https://localhost:9443/
```

Back to top
