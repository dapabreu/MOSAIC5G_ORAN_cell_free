# Introduction

Portainer is an open-source management tool used for the orchestration of containerized applications, in the form of a web dashboard. The portainer itself is instantiated as a container that works as a Virtual Appliance.

In the context of 5G and Mosaic5G/OpenAirInterface, this tool can be valuable in order to more easily and efficiently manage all of the containerized applications. For our use case, Portainer can help address challenges associated with using a command-line-based operating system like Ubuntu Server

# Implementation

## Tutorial followed

[Install Portainer w/ Docker on Linux | Portainer Documentation](https://docs.portainer.io/start/install-ce/server/docker/linux)


## Pre-requisites

Firstly, make sure the Docker is running and fully functional: `docker --version`

Additionally, the docker-compose plugin by running: `docker-compose`

Expected output should be: 

```bash
# for docker
Docker version 24.0.5, build ced0996
# for cocker-compose
Docker Compose version v2.20.2-desktop.1
```
If not, please go to: [Docker Installation Guides](https://docs.docker.com/engine/install/)

## Deployment

Normally, **Portainer** is implemented directly on the hosts OSs with a single command:

```bash
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:2.21.4
```
| Argument                | Description                                                                                          |
|-------------------------|------------------------------------------------------------------------------------------------------|
| `docker run`            | Command to create and run a new container.                                                          |
| `-d`                    | Runs the container in detached mode (in the background).                                            |
| `-p 8000:8000`          | Maps port 8000 on the host to port 8000 in the container - TCP tunnel server only used for specific purposes                 |
| `-p 9443:9443`          | Maps port 9443 on the host to port 9443 in the container - used for the secure Portainer web interface |
| `--name portainer`      | Assigns the name `portainer` to the container.                                                      |
| `--restart=always`      | Ensures the container automatically restarts if it stops or if the host is rebooted.                |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Mounts the Docker socket from the host to the container, enabling Portainer to manage Docker resources directly. |
| `-v portainer_data:/data` | Creates and mounts a volume named `portainer_data` to persist Portainer's configuration and state. |
| `portainer/portainer-ce:2.21.4` | Specifies the Portainer Community Edition (CE) image                   |

In our case, and to better expedite the process we used a `docker-compose.yml` file - [*Source*](https://earthly.dev/blog/portainer-for-docker-container-management/)

```yml
version: "3"
services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - 9005:9443
     volumes:
       - data:/data
       - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
volumes:
  data:
```
> *Note: the internal port bind number is changed due to firewall restrictions set in place by the XenOrchestra Virtual Appliance*

## Running

In order to run the container make sure you are located in the same folder as the `docker-compose.yml` file and run: `docker-compose up -d`.