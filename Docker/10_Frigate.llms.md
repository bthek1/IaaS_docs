# Frigate

Frigate

Author

Benedict Thekkel

## Prerequisites

- Proxmox VE installed
- LXC container (Ubuntu/Debian recommended)
- Coral TPU or other hardware acceleration (optional but recommended)

------------------------------------------------------------------------

## Setup Steps

**1. Create LXC Container** - Use Debian 12 or Ubuntu 22.04 template - Enable “Nesting” and “FUSE” in container options - Set resources: 2+ cores, 4GB+ RAM - For Coral TPU: make container privileged or pass through USB device

**2. Install Docker in Container**

``` bash
# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

**3. Create Frigate Directory Structure**

``` bash
mkdir -p /opt/frigate/config
mkdir -p /opt/frigate/storage
```

**4. Create Basic Frigate Config** Create `/opt/frigate/config/config.yml`:

``` yaml
mqtt:
  enabled: false

cameras:
  dummy_camera:
    enabled: false
    ffmpeg:
      inputs:
        - path: rtsp://your_camera_ip:554/stream
          roles:
            - detect
```

**5. Run Frigate Container**

``` bash
docker run -d \
  --name frigate \
  --restart=unless-stopped \
  --mount type=tmpfs,target=/tmp/cache,tmpfs-size=1000000000 \
  --device /dev/bus/usb:/dev/bus/usb \
  --shm-size=64m \
  -v /opt/frigate/storage:/media/frigate \
  -v /opt/frigate/config:/config \
  -v /etc/localtime:/etc/localtime:ro \
  -p 5000:5000 \
  -p 8554:8554 \
  -p 8555:8555/tcp \
  -p 8555:8555/udp \
  ghcr.io/blakeblackshear/frigate:stable
```

**6. Access Frigate** - Web UI: `http://CONTAINER_IP:5000`

------------------------------------------------------------------------

## For Coral TPU Support

In Proxmox host, pass USB device to container:

``` bash
# Find Coral device
lsusb

# Edit container config
nano /etc/pve/lxc/CTID.conf

# Add lines:
lxc.cgroup2.devices.allow: c 189:* rwm
lxc.mount.entry: /dev/bus/usb/001 dev/bus/usb/001 none bind,optional,create=dir
```

Need help with a specific step or want to configure cameras?

------------------------------------------------------------------------

Back to top
