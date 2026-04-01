---
title: Overleaf-LaTeX
description: 
published: true
date: 2026-04-01T15:11:31.213Z
tags: docker, latex
editor: markdown
dateCreated: 2026-03-16T13:51:51.795Z
---

### Introduction

Overleaf is a Software with that you can edit LaTeX files in a browser. This is a Tutorial on how to setup a self-hosted Overleaf Community Edition server on Linux.

It is intended that you **run** this **server** **behind a reverse proxy**. It is **very** **important** to do so **if** **you** plan to **publish** the **application** **to the internet!**

### Preparation

- Linux server
- git & docker
- Reverse proxy (not included in this tutorial)

``` bash
# Get packages with the package manager of your distro

sudo dnf install git

## dnf RHEL based (RHEL, Rocky, Alma, Centos, etc.)
## apt Debian based (Debian, Ubuntu, Mint, etc.)
## and so on

# add dedicated application user
adduser overleaf

# Be sure that your designated user is part of the docker group
## Give application user right for docker
usermod -aG docker overleaf

## Check it when you are logged in as overleaf - displays docker
groups
```

**<u>!Caution!</u>**

*When using RHEL based os for the server it is recomended to deinstall Podman and install Docker CE. Podman was not tested.*

### Installation

*Sources:
* [Official](https://github.com/overleaf/toolkit/blob/master/doc/quick-start-guide.md) 
* [Additional Instructions](https://www.scaleway.com/en/docs/tutorials/overleaf/)* 
* [Example Implementation](https://blog.felixviola.de/overleaf-ce-self-host-your-own-latex-server-tutorial/) 
* [Completing Tex Live](https://github.com/overleaf/toolkit/blob/master/doc/ce-upgrading-texlive.md) 
* [Ubuntu 20.04 TexLive full](https://docs.vultr.com/how-to-install-overleaf-community-edition-on-ubuntu-20-04-lts) 
* [Old scheme full](https://github.com/overleaf/overleaf/wiki/Quick-Start-Guide)

**Installation successfully tested on the 19.07.2024**

``` bash
# Make a directory and cahnge the permissions
sudo mkdir /opt/overleaf-toolkit
sudo chown overleaf:root /opt/overleaf-toolkit

# change to application user
su - overleaf

# Download git repo to opt and cd into it
git clone https://github.com/overleaf/toolkit.git /opt/overleaf-toolkit
cd /opt/overleaf-toolkit

# initialize the project
./bin/init

## (OPTIONAL) change any variable needed in this directory
ls config
vi config/overleaf.rc

----------------START-----------------

## OVERLEAF_LISTEN_IP=127.0.0.1
OVERLEAF_LISTEN_IP=<IP_OF_VM>

## SIBLING_CONTAINERS_ENABLED=true
SIBLING_CONTAINERS_ENABLED=false

----------------STOP------------------

# Add the http service to the local firewall
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

# Test if you can start the application
## This takes some time
./bin/up

# Check if the website is reachable


# Stop the application if needed
./bin/stop
```

### Configuration

``` bash
# Make the application to a systemd service
vi /etc/systemd/system/overleaf.service

----------------START-----------------
[Unit]
Description=Overleaf Community Edition

[Service]
Type=oneshot
WorkingDirectory=/opt/overleaf-toolkit
ExecStart=/opt/overleaf-toolkit/bin/up -d
ExecStop=/opt/overleaf-toolkit/bin/stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
----------------STOP------------------

# Reload the deamon, enable & start overleaf Service
systemctl daemon-reload
systemctl enable overleaf.service
systemctl start overleaf.service

# Check status of service
systemctl status overleaf.service
```

#### Tex Live full install {#tex_live_full_install}

By default Overleaf Community Edition installs LaTeX in a \"lite\" version. In order to use the full capabilities of Overleaf it is advised to fully install LaTex (all packages).

> *Remember, that once you update the image used for Overleaf you need to redo this step. Each time. That\'s CE Version for you :)*

``` bash
# Change in to working directory
cda

# Enter a shell into the container
./bin/shell

# Current container still has TeX Live 2025 so no longer update. Change repo
tlmgr option repository https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/2025/tlnet-final

# Update TeX Live Manager
tlmgr update --self

## Display version of TeX Live
tlmgr --version
tlmgr revision 71331 (2024-05-24 09:30:36 +0200)
TeX Live (https://tug.org/texlive) version 2024

# Install full scheme (takes about 10 to 20 min)
tlmgr install scheme-full

# Exit container shell
exit
```

### Update

``` bash
# Stop service
systemctl stop overleaf.service

# Change into working directory
cda

# Update
./bin/upgrade

# Stop automaticly started containers
./bin/stop

# Start service again
systemctl start overleaf.service

## Source: https://github.com/overleaf/toolkit/blob/master/doc/upgrading.md
```
