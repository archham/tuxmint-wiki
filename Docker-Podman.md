---
title: Docker-Podman
description: 
published: true
date: 2026-03-17T16:43:14.679Z
tags: docker, podman, container
editor: markdown
dateCreated: 2026-03-16T13:50:36.723Z
---

## Introduction

Usefull info to Docker and Podman.

## Commands

All commands can be run with either docker or podman.

``` bash
#List running Docker containers
docker ps

#List all Docker containers
docker ps -a

#Pull new images
cd /directory/of/docker-compose/file
docker-compose pull

#Start up persitant with docker compose file
docker-compose up -d

#Stop container(s) from docker compose file
docker-compose down

#List all Docker images
docker container ls -a

#Remove Docker images
docker container rm <container id>

#Login to docker.io
docker login --username <USERNAME>

#Open a shell in docker
docker exec -it <container-name-or-id> <shell-executable>
##Example
docker exec -it c4d3132c4212 bash
```

## Compose

Docker compose is a very useful method to run one or more containers repeatedly in a controlled manner even with dependencies.

### Selinux

If you use `selinux` you can add the `z` or `Z` options to modify the selinux label of the host file or directory being mounted into the container. This affects the file or directory on the host machine itself and can have consequences outside of the scope of Docker.

- The `z` option indicates that the bind mount content is shared among multiple containers.
- The `Z` option indicates that the bind mount content is private and unshared.

Use extreme caution with these options. Bind-mounting a system directory such as `/home` or `/usr` with the `Z` option renders your host machine inoperable and you may need to relabel the host machine files by hand.

> When using bind mounts with services, selinux labels (`:Z` and `:z`),
> as well as `:ro` are ignored. See [moby/moby #32579](https://github.com/moby/moby/issues/32579) for details.

This example sets the `z` option to specify that multiple containers can share the bind mount\'s contents:

It is not possible to modify the selinux label using the `--mount` flag. [1](https://docs.docker.com/engine/storage/bind-mounts/#configure-the-selinux-label) [2](https://github.com/containers/podman/issues/10779) [3](https://github.com/containers/podman-compose/issues/246) [4](https://unix.stackexchange.com/questions/651198/podman-volume-mounts-when-to-use-the-z-or-z-suffix)

``` bash
docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app:z \
  nginx:latest
```

## Networking Docker

### Change default networking

Make sure to define a default network for the bridge mode, otherwise you may run into trouble of already used networks

``` bash
ip route show #check networks

systemctl stop docker
vi /etc/docker/daemon.json

Configure Docker Daemon:

Edit the Docker daemon configuration file, typically located at /etc/docker/daemon.json. 
If this file does not exist, you can create it.

You'll need to specify an alternative subnet for Docker to use for its bridge networks. 
For example, if you want to use 192.168.0.0/16 for Docker, you can add the following configuration:
```

``` json
{
  "bip": "192.168.0.1/24",
  "default-address-pools": [
    {"base":"192.168.0.0/16","size":24}
  ]
}
```

  **Options dockerd (Docker daemon)** [External Link](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file)
  |Option                                 | Detail                                                |
  |---------------------------------------|-------------------------------------------------------|
  |--bip string                           |Specify network bridge IP                              |
  |--default-address-pool pool-options    |Default address pools for node specific local networks |

``` bash
systemctl start docker
docker network ls #verify settings
docker network inspect docker_default
```

## Networking Podman

### Change default networking

Container engines like Podman & Buildah read **containers.conf** file, if it exists and modify the defaults for running containers on the host. containers.conf uses a TOML format that can be easily modified and versioned.

Container engines read the **/usr/share/containers/containers.conf**, **/etc/containers/containers.conf**, and **/etc/containers/containers.conf.d/\*.conf** for global configuration
that effects all users.[\[External Source\]](https://github.com/containers/common/blob/main/docs/containers.conf.5.md) [5](https://docs.podman.io/en/latest/markdown/podman-network.1.html#podman-network) [6](https://github.com/containers/podman/blob/main/docs/tutorials/basic_networking.md)

*Note, container engines also use other configuration files for configuring the environment.*

- storage.conf for configuration of container and images storage.
- registries.conf for definition of container registries to search while pulling. container images.
- policy.conf for controlling which images can be pulled to the system.
```bash
ip route show #check networks

systemctl stop podman vi /usr/share/containers/containers.conf

default_subnet=\"10.88.0.0/16\"

The subnet to use for the default network (named above in
default_network). If the default network does not exist, it will be
automatically created the first time a tool is run using this subnet.

default_subnet_pools=\[\]

DefaultSubnetPools is a list of subnets and size which are used to
allocate subnets automatically for podman network create. It will
iterate through the list and will pick the first free subnet with the
given size. This is only used for ipv4 subnets, ipv6 subnets are always
assigned randomly.

The default list is (10.89.0.0-10.255.255.0/24):
```

``` bash
default_subnet_pools = [
  {"base" = "10.89.0.0/16", "size" = 24},
  {"base" = "10.90.0.0/15", "size" = 24},
  {"base" = "10.92.0.0/14", "size" = 24},
  {"base" = "10.96.0.0/11", "size" = 24},
  {"base" = "10.128.0.0/9", "size" = 24},
]
```

``` bash
systemctl start podman
podman network ls #verify settings
podman network inspect podman
```
