# Docker Jupyter Labs

Docker Jupyter Labs

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  jupyterlab:
    image: jupyter/base-notebook:latest
    container_name: jupyterlab
    ports:
      - "888:8888"
    volumes:
      - ./notebooks:/home/jovyan/work
    environment:
      - JUPYTER_TOKEN=3719  # Set a secure token
      - NB_USER=ben
      - NB_UID=1000       # Replace with your UID
      - NB_GID=1000       # Replace with your GID
    restart: unless-stopped
```

## To Run

``` sh
docker-compose up -d
```

## URL

``` sh
http://localhost:888
```

## Check Permission

``` sh
id bthek1
```

## Password Hash

``` sh
python -c "from jupyter_server.auth import passwd; print(passwd('1234', 'sha1'))"
```

Back to top
