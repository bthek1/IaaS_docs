# Influx Docker

Influx Docker

Author

Benedict Thekkel

## docker-compose.yml

``` yml
services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=adminpassword
      - DOCKER_INFLUXDB_INIT_ORG=org
      - DOCKER_INFLUXDB_INIT_BUCKET=bucket
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=token
      - INFLUXD_HTTP_BIND_ADDRESS=:8086
      - INFLUXD_HTTP_V1_COMPATIBILITY_API=true
      - INFLUXD_HTTP_AUTH_ENABLED=true
    restart: unless-stopped

volumes:
  influxdb_data:
```

Back to top
