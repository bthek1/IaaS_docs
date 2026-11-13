# Docker Overview

Core Docker concepts: images, containers, volumes, and how a Dockerfile is layered.

Author

Benedict Thekkel

## Definitions

- Dockerfile: Blueprint for building a docker image

- compose.yml: Builds and runs multiple docker images

- requirements.txt: pip install libraries list

- Image: Template for running a docker containers

- Container: Running process

## Docker File

`FILE _image_` - base image ex. ubuntu, node:12

`WORKDIR _filename_`- like when you cd into a directory

***every step from now is considered layes***

`COPY _filename_ _location_`- ex. COPY package\*.json ./

`RUN _npm install_` - exec commands - shell form

`COPY . .` - copy all the file in directory into the container

`ENV _PORT=8080_` -

`EXPOSE _port_` - exposed port ex. 8080

`CMD ["npm" "start"]` - only one of these in the file. exec form

### Example: pytorch image

``` dockerfile
# Base image with PyTorch and GPU support
FROM pytorch/pytorch:latest

# Install additional packages
RUN apt-get update && \
    apt-get install -y sudo software-properties-common

RUN add-apt-repository -y ppa:bashtop-monitor/bashtop && \
    add-apt-repository -y ppa:flexiondotorg/nvtop && \
    apt-get update

RUN apt-get install -y bashtop nvtop python3-pip \
    vim poppler-utils git openssh-client && \
    apt-get clean

# Add user 'ben'
RUN useradd -m -s /bin/bash ben && \
    echo "ben ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Set working directory
WORKDIR /home/ben

# Switch to user ben
USER ben

# Add /home/ben/.local/bin to the PATH
ENV PATH="/home/ben/.local/bin:${PATH}"

# Copy requirements file
COPY requirements.txt .

# Install required Python packages
RUN pip install --no-cache-dir -r requirements.txt

RUN git config --global user.email <email> \
    && git config --global user.name "main_docker"

# COPY /home/ben/'.jupyter' '.jupyter'
ADD /apputils-extension /home/ben/.jupyter/lab/user-settings/@jupyterlab/apputils-extension

# Expose Jupyter notebook port
EXPOSE 8888

# Command to start JupyterLab with dark theme
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--NotebookApp.theme='JupyterLab Dark'", "--NotebookApp.token='test'"]
```

### .dockerignore

> just like a git ignore file

### Building Docker image

``` sh
docker build -t _name_ _path_
```

ex.

``` sh
docker build -t test:taggedv0.1
```

-t : tag

### View Docker Images

``` sh
docker images
```

### Delete Image

``` sh
docker rmi _Image_
```

### Running Docker image as Container

``` sh
docke run -p 5000:8080 _name_
```

``` sh
docker run -p 8888:8888 -td --gpus=all --name testcontainer test
```

### View Dockers Containers Running

``` sh
docker ps -a
```

### Deleting Docker Containers

``` sh

docker rmi _image_
```

### Stop Docker Running

``` sh
docker stop $(docker ps -aq)
```

## Volumes

> Folder which multiple containers can access

------------------------------------------------------------------------

``` sh
docker volume create _name_
```

### Using volumes in container

``` sh
docker run --mount source=_volumename_, target = _location_
```

## Docker Debugging

- Inspect logs in docker desktop

- docker exec

## TIPs

- each docker should only do one task

- Done through docker-compose.yml

## Compose.yml

### Example

> this [GPU](https://docs.docker.com/compose/gpu-support/)

> [nvidia Docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/1.14.5/install-guide.html)

``` yml
services:
  torch:
    container_name: main_docker
    
    build:
      context: .

    volumes:
      - /home/ben/BENEDICT_Only:/home/ben/BENEDICT_Only
      
    ports:
      - "8888:8888"
    
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

``` yml
version: '3'
services:
    web: 
        build: .
        ports:
            - "8080:8080"
    db: 
        image: "mysql"
        environment:
            MYSQL_ROOT_PASSWORD: password
        volumes:
            - db-data:/foo
            
volumes:
    db-data:
            
```

### Running Docker Compose

``` sh
docker-compose up
```

### Stopping Docker Compose

``` sh
docker-compose down
```

## requirements.txt

### Example

``` txt
jupyterlab
fastai
fastbook
timm
pdf2image
pandas
nbdev
nbdevAuto
```

## Docker Network Types

> Docker containers can use several types of networks to communicate with each other, the host, or external networks. Here’s a summary of the main Docker network types:

### 1. **Bridge Network (Default)**

- **Description**: This is the default network type for Docker containers. When you run a container without specifying a network, it is attached to the `bridge` network.

- **Use Case**: Ideal for communication between multiple containers on the same Docker host but isolated from the host’s network.

- **Example**:

  ``` bash
  docker network create my_bridge_network
  docker run --network=my_bridge_network my_container
  ```

### 2. **Host Network**

- **Description**: The container shares the host’s network stack, meaning it has direct access to the host’s network interfaces (no isolation).

- **Use Case**: Useful when you want to avoid the network isolation provided by Docker, such as when running services that need high network performance or direct access to host network services.

- **Example**:

  ``` bash
  docker run --network=host my_container
  ```

- **Note**: Not available on Windows containers.

### 3. **None Network**

- **Description**: The container has no network interfaces, except for the loopback interface (`localhost`).

- **Use Case**: Useful for standalone containers that don’t need network connectivity.

- **Example**:

  ``` bash
  docker run --network=none my_container
  ```

### 4. **Overlay Network**

- **Description**: This network allows containers running on different Docker hosts to communicate with each other. It is commonly used with Docker Swarm or Kubernetes.

- **Use Case**: Useful for deploying a multi-host Docker network (e.g., in a swarm or Kubernetes cluster).

- **Example**:

  ``` bash
  docker network create --driver overlay my_overlay_network
  docker service create --network=my_overlay_network my_service
  ```

### 5. **Macvlan Network**

- **Description**: This network type allows you to assign a MAC address to a container, making it appear as a physical device on your network. The container gets a unique IP address on the local network.

- **Use Case**: Useful for running containers that need to appear directly on the physical network and communicate with other physical devices (e.g., running legacy applications).

- **Example**:

  ``` bash
  docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 my_macvlan_network

  docker run --network=my_macvlan_network my_container
  ```

### 6. **Custom User-Defined Bridge Network**

- **Description**: Similar to the default `bridge` network but user-defined networks allow for better control and communication between containers. Containers on the same custom bridge network can communicate by name, and it isolates containers from others outside the network.

- **Use Case**: Recommended for multi-container applications where containers need to communicate with each other (e.g., a web service with a separate database).

- **Example**:

  ``` bash
  docker network create my_bridge_network
  docker run --network=my_bridge_network my_container
  ```

### 7. **IPvlan Network**

- **Description**: Similar to `macvlan`, but more lightweight and allows you to manage how your containers connect to the underlying network by operating at the network layer (Layer 3).

- **Use Case**: Useful for containers that need to communicate directly with external networks.

- **Example**:

  ``` bash
  docker network create -d ipvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 my_ipvlan_network

  docker run --network=my_ipvlan_network my_container
  ```

### 8. **Ingress Network (Swarm Only)**

- **Description**: The default network for services in Docker Swarm mode. It is used for load balancing services across the swarm.

- **Use Case**: Automatically created and used when you initialize Docker Swarm. It’s useful for scaling services and distributing them across multiple hosts.

- **Example**:

  ``` bash
  docker service create --name my_service --network ingress my_image
  ```

### Summary of Network Types:

| Network Type | Isolation | Multi-host | Use Case |
|----|----|----|----|
| Bridge (Default) | Yes | No | Communication between containers on the same host |
| Host | No | No | High performance, direct access to host network |
| None | N/A | N/A | Containers that don’t need network connectivity |
| Overlay | Yes | Yes | Multi-host communication (e.g., Docker Swarm, Kubernetes) |
| Macvlan | No | Yes | Containers appear as devices on the physical network |
| IPvlan | No | Yes | Lightweight alternative to macvlan for Layer 3 connectivity |
| Custom Bridge | Yes | No | Enhanced control over container networking |
| Ingress (Swarm) | Yes | Yes | Swarm load balancing |

Let me know if you need further details on any of these!

Back to top
