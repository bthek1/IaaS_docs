# Docker Firefox

Docker Firefox

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  firefox:
    image: lscr.io/linuxserver/firefox:latest
    container_name: firefox
    environment:
      - PUID=1000            # User ID of your host user
      - PGID=1000            # Group ID of your host user
      - TZ=Etc/UTC           # Timezone, e.g., "America/New_York"
      - WEBPAGE=https://www.mozilla.org  # Starting URL
      - VNC_PASSWORD=3719
      - SUBFOLDER=/firefox  # Access at http://yourdomain.com/firefox
      - FIREFOX_CLI=https://www.firefox.io/ #optional
      - MOZ_DISABLE_HW_ACCEL=1   # Disable hardware acceleration

    volumes:
      - ./config:/config     # Persist configuration data
    ports:
      - 3000:3000            # Port mapping
    shm_size: "2gb"          # Shared memory size
    restart: unless-stopped
    devices:
      - /dev/dri:/dev/dri
```

## To Run

``` sh
docker-compose up -d
```

## URL

``` sh
http://localhost:3000
```

Back to top
