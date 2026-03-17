---
title: Changedetection
description: 
published: true
date: 2026-03-17T16:03:20.087Z
tags: docker
editor: markdown
dateCreated: 2026-03-16T13:50:27.590Z
---

![Changedetection Logo](Changedetection_Logo.png "Changedetection Logo")

## Introduction

Changedetection.io is a usefull tool to track changes in websites. For
example software versions, etc.

## Setup with Docker compose {#setup_with_docker_compose}

``` yaml
# cat changedetection.io/docker-compose.yml 
version: '3.2'
services:
    changedetection:
      image: ghcr.io/dgtlmoon/changedetection.io
      container_name: changedetection
      hostname: changedetection
      volumes:
        - changedetection-data:/datastore
# Configurable proxy list support, see https://github.com/dgtlmoon/changedetection.io/wiki/Proxy-configuration#proxy-list-support
#        - ./proxies.json:/datastore/proxies.json

  #    environment:
  #        Default listening port, can also be changed with the -p option
  #      - PORT=5000

  #      - PUID=1000
  #      - PGID=1000
  #
  #       Alternative WebDriver/selenium URL, do not use "'s or 's!
  #      - WEBDRIVER_URL=http://browser-chrome:4444/wd/hub
  #
  #       WebDriver proxy settings webdriver_proxyType, webdriver_ftpProxy, webdriver_noProxy,
  #                                webdriver_proxyAutoconfigUrl, webdriver_autodetect,
  #                                webdriver_socksProxy, webdriver_socksUsername, webdriver_socksVersion, webdriver_socksPassword
  #
  #             https://selenium-python.readthedocs.io/api.html#module-selenium.webdriver.common.proxy
  #
  #       Alternative Playwright URL, do not use "'s or 's!
  #      - PLAYWRIGHT_DRIVER_URL=ws://playwright-chrome:3000/?stealth=1&--disable-web-security=true
  #
  #       Playwright proxy settings playwright_proxy_server, playwright_proxy_bypass, playwright_proxy_username, playwright_proxy_password
  #
  #             https://playwright.dev/python/docs/api/class-browsertype#browser-type-launch-option-proxy
  #
  #        Plain requests - proxy support example.
  #      - HTTP_PROXY=socks5h://10.10.1.10:1080
  #      - HTTPS_PROXY=socks5h://10.10.1.10:1080
  #
  #        An exclude list (useful for notification URLs above) can be specified by with
  #      - NO_PROXY="localhost,192.168.0.0/24"
  #
  #        Base URL of your changedetection.io install (Added to the notification alert)
  #      - BASE_URL=https://mysite.com
  #        Respect proxy_pass type settings, `proxy_set_header Host "localhost";` and `proxy_set_header X-Forwarded-Prefix /app;`
  #        More here https://github.com/dgtlmoon/changedetection.io/wiki/Running-changedetection.io-behind-a-reverse-proxy-sub-directory
  #      - USE_X_SETTINGS=1
  #
  #        Hides the `Referer` header so that monitored websites can't see the changedetection.io hostname.
  #      - HIDE_REFERER=true
  #        
  #        Default number of parallel/concurrent fetchers
  #      - FETCH_WORKERS=10

      # Comment out ports: when using behind a reverse proxy , enable networks: etc.
      ports:
        - 5000:5000
      restart: unless-stopped

     # Used for fetching pages via WebDriver+Chrome where you need Javascript support.
     # Now working on arm64 (needs testing on rPi - tested on Oracle ARM instance)
     # replace image with seleniarm/standalone-chromium:4.0.0-20211213
     
     # If WEBDRIVER or PLAYWRIGHT are enabled, changedetection container depends on that
     # and must wait before starting (substitute "browser-chrome" with "playwright-chrome" if last one is used)
#    depends_on:
#        browser-chrome:
#            condition: service_started

#    browser-chrome:
#        hostname: browser-chrome
#        image: selenium/standalone-chrome-debug:3.141.59
#        environment:
#            - VNC_NO_PASSWORD=1
#            - SCREEN_WIDTH=1920
#            - SCREEN_HEIGHT=1080
#            - SCREEN_DEPTH=24
#        volumes:
#            # Workaround to avoid the browser crashing inside a docker container
#            # See https://github.com/SeleniumHQ/docker-selenium#quick-start
#            - /dev/shm:/dev/shm
#        restart: unless-stopped

     # Used for fetching pages via Playwright+Chrome where you need Javascript support.
     # Note: Playwright/browserless not supported on ARM type devices (rPi etc)
#    playwright-chrome:
#        hostname: playwright-chrome
#        image: browserless/chrome
#        restart: unless-stopped
#        environment:
#            - SCREEN_WIDTH=1920
#            - SCREEN_HEIGHT=1024
#            - SCREEN_DEPTH=16
#            - ENABLE_DEBUGGER=false
#            - PREBOOT_CHROME=true
#            - CONNECTION_TIMEOUT=300000
#            - MAX_CONCURRENT_SESSIONS=10
#            - CHROME_REFRESH_TIME=600000
#            - DEFAULT_BLOCK_ADS=true
#            - DEFAULT_STEALTH=true
#
#             Ignore HTTPS errors, like for self-signed certs
#            - DEFAULT_IGNORE_HTTPS_ERRORS=true
#
volumes:
  changedetection-data:
```

## Local backup {#local_backup}

Local backup of Changedetection with cron twice a day on docker.

``` bash
## crontab implementation
# crontab -l
15 12,21 * * * /srv/changedetection_backup/bin/changedetection_backup.sh >/dev/null
```

``` bash
## backup sript
# cat /srv/changedetection_backup/bin/changedetection_backup.sh 

#/bin/bash
# DESC: Backup changedetection volume and keep X generations

GEN=60
STORAGE_BASE=/var/lib/docker/volumes/changedetectionio_changedetection-data
DOCKER_VOLUME_PATTERN=_data
BACKUP_DIR=/srv/changedetection_backup/data

# Use the base storage path and pattern to find the matching data directory
DOCKER_VOLUME_DIR=$(find $STORAGE_BASE -type d -name "$DOCKER_VOLUME_PATTERN" -print -quit)

# Check if the data directory was found
if [ -z "$DOCKER_VOLUME_DIR" ]; then
    echo "ERROR: No directory found matching pattern $DOCKER_VOLUME_PATTERN in $STORAGE_BASE"
    exit 1
fi

# Perform the backup
tar cfz $BACKUP_DIR/changedetection_$(date '+%Y%m%d%H%M').tar.gz "$DOCKER_VOLUME_DIR" 2>&1

# Rotate the backups: keep only the newest $GEN backups
# Sort backups by date, delete older ones beyond the $GEN count
find $BACKUP_DIR -type f -name 'changedetection_*.tar.gz' | sort | head -n -${GEN} | xargs -r rm -fv

# List backups
ls -l $BACKUP_DIR
```
