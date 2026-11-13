# Install Docker

Adding Docker’s apt repository on Linux, installing the engine, and running it as a non-root user.

Author

Benedict Thekkel

## Install Docker on Linux

### Set up Docker’s apt repository.

``` python
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Install the Docker packages.

``` sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify the Installation

> Check if Docker is installed correctly by running

``` sh
sudo docker --version
```

### Docker Check

> Verify that the Docker Engine installation is successful by running the hello-world image.

``` sh
sudo docker run hello-world
```

## Manage Docker as a Non-Root User (Optional)

### Create the docker group.

``` sh
sudo groupadd docker
```

### Add your user to the docker group.

``` sh
sudo usermod -aG docker $USER
```

### You can also run the following command to activate the changes to groups:

``` sh
newgrp docker
```

## Configure Docker to start on boot with systemd

``` sh
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

``` sh
sudo systemctl enable docker
```

### To stop this behavior, use disable instead.

``` sh
sudo systemctl disable docker.service
sudo systemctl disable containerd.service
```

## Uninstall Docker

### Uninstall the Docker Engine, CLI, containerd, and Docker Compose packages:

``` sh
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
```

### Removing Images, containers, volumes

> Images, containers, volumes or custom configuration files on your host aren’t automatically removed. To delete all images, containers, and volumes:

``` sh
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

## Using Docker Compose

### Install Docker Compose

``` sh
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

or

``` sh
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Create a `docker-compose.yml` File

> Create a `docker-compose.yml` file in your project directory

``` sh
version: '3'
services:
  web:
    image: nginx
    ports:
      - "80:80"
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: example
```

### Run Docker Compose

``` sh
docker-compose up
```

## Managing Docker Volumes and Networks

> Volumes: Used to persist data outside of containers

``` sh
docker volume create my_volume
docker run -d -v my_volume:/data ubuntu
```

> Networks: Used to allow containers to communicate

``` sh
docker network create my_network
docker run -d --network=my_network --name=container1 ubuntu
docker run -d --network=my_network --name=container2 ubuntu
```

## Clean Up Unused Docker Resources

> Over time, Docker might accumulate unused resources. You can clean them up with

------------------------------------------------------------------------

``` sh
docker system prune -a
```

Back to top
