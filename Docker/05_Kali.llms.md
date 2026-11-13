# Docker Kali

[linux server](https://fleet.linuxserver.io/image?name=linuxserver/kali-linux)

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  kali-linux:
    image: lscr.io/linuxserver/kali-linux:latest
    container_name: kali-linux
    security_opt:
      - seccomp:unconfined #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - SUBFOLDER=/ #optional
      - "TITLE=Kali Linux" #optional
    volumes:
      - /path/to/data:/config
      - /var/run/docker.sock:/var/run/docker.sock #optional
    ports:
      - 3000:3000
      - 3001:3001
    devices:
      - /dev/dri:/dev/dri #optional
    shm_size: "1gb" #optional
    restart: unless-stopped
```

## To Run

``` sh
docker-compose up -d
```

## URL

``` sh
http://localhost:3000/
```

## Check Permission

``` sh
id bthek1
```

Back to top
