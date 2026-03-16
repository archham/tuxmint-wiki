---
title: AdGuardHome
description: 
published: true
date: 2026-03-16T22:24:37.954Z
tags: dns, linux
editor: markdown
dateCreated: 2026-03-16T13:50:13.451Z
---

AdGuard Home is a way of keeping your browser experience pleasant and to protect you from unwanted trackers.

## Preparation

This will be a Guide on how to run AdGuard Home on Rocky Linux 9 with
selinux enabled.

1.  Setup a Rocky Linux 9 server
2.  Install the software
3.  Configure it

## Installation

``` bash
# ssh into your server,become root and enable selinux
setenforce 1
getenforce #check the status

# change directory
cd /opt/

# download and install the software
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# add firewall rules
firewall-cmd --new-zone=adguard --permanent
firewall-cmd --zone=adguard --add-source=192.168.0.0/24 --permanent
firewall-cmd --zone=adguard --add-port=3000/tcp --permanent
firewall-cmd --zone=adguard --add-port=53/udp --permanent
firewall-cmd --zone=adguard --add-port=80/tcp --permanent
firewall-cmd --reload

# visit configuration website (listed in installation output) in browser
...
AdGuard Home is successfully installed and will automatically start on boot.
There are a few more things that must be configured before you can use it.
Click on the link below and follow the Installation Wizard steps to finish setup.
AdGuard Home is now available at the following addresses:
2024/10/25 13:17:24 [info] go to http://127.0.0.1:3000
2024/10/25 13:17:24 [info] go to http://[::1]:3000
2024/10/25 13:17:24 [info] go to http://192.168.0.198:3000
2024/10/25 13:17:24 [info] go to http://[fe80::be24:11ff:fe90:3f62%ens18]:3000
...

# additional info
...
AdGuard Home is now installed and running
you can control the service status with the following commands:
sudo /opt/AdGuardHome/AdGuardHome -s start|stop|restart|status|install|uninstall
...
```

## Configuration

tbd

<references />

[^1]: <https://github.com/AdguardTeam/AdguardHome>

[^2]: <https://adguard.com/en/adguard-home/overview.html>
