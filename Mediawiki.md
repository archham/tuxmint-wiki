---
title: Mediawiki
description: 
published: true
date: 2026-03-16T13:57:05.593Z
tags: docker
editor: markdown
dateCreated: 2026-03-16T13:51:37.465Z
---

## Introduction

This will show you how you can run MediaWiki on Docker.

Created by the following
[Youtube](https://www.youtube.com/watch?v=D4cF9rEhXuk) video and
[instructions](https://i12bretro.github.io/tutorials/0594.html). Big
thank you to [i12bretro](https://www.youtube.com/@i12bretro) STILL UNDER
CONSTRUCTION

## Docker Compose File {#docker_compose_file}

``` yaml
version: '1.0'
services:
  mariadb:
    image: mariadb:latest
    container_name: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: {SECRET1}
      MYSQL_USER: mediawiki_rw
      MYSQL_PASSWORD: {SECRET2}
      MYSQL_DATABASE: mediawiki
    volumes:
      - /home/${USER}/docker/mariadb:/var/lib/mysql
    networks:
      - containers
    restart: always

  mediawiki:
    image: mediawiki:latest
    container_name: mediawiki
    depends_on:
      - mariadb
    ports:
      -  8080:80
    environment:
      MYSQL_USER: mediawiki_rw
      MYSQL_PASSWORD: {SECRET2}
      MYSQL_DATABASE: mediawiki
    volumes:
      - /home/${USER}/docker/mediawiki/images:/var/www/html/images
      - /home/${USER}/docker/mediawiki/LocalSettings.php:/var/www/html/LocalSettings.php
      - /home/${USER}/docker/mediawiki/logo_mediawiki.png:/var/www/html/resources/assets/logo_mediawiki.png
    networks:
      - containers
    restart: always

networks:
  containers:
    driver: bridge
```

## Configuration

### Add category to sidebar {#add_category_to_sidebar}

Enter MediaWiki:Sidebar in searchbox and then edit the page:

``` mediawiki

* navigation
** mainpage|mainpage-description
** recentchanges-url|recentchanges
** randompage-url|randompage
** helppage|help-mediawiki
** Category:Linux|Linux
* SEARCH
* TOOLBOX
* LANGUAGES
```

## Mediawiki Dump {#mediawiki_dump}

How to save Mediawiki\'s easily.

### Media Wiki Dump Generator {#media_wiki_dump_generator}

``` bash
#Get dependencies
dnf install git python3

#Get Poetry for Python3 (Source: https://github.com/python-poetry/install.python-poetry.org)
curl -sSL https://install.python-poetry.org | python3 -

#Clone git repo
git clone https://github.com/mediawiki-client-tools/mediawiki-dump-generator

#Change directory and install the application
cd mediawiki-dump-generator
poetry update && poetry install && poetry build
pip3 install --force-reinstall dist/*.whl

#Use the application 

##Get the engine of the Wiki
dumpgenerator https://wiki.rueti16.com --get-wiki-engine
##Get the content of the Wiki with the images
dumpgenerator https://wiki.rueti16.com --xml --images
##Get the content of the Wiki with the help of the API
dumpgenerator --api https://wiki.rueti16.com/api.php --xml --images

##More infos @ https://github.com/mediawiki-client-tools/mediawiki-dump-generator/blob/python3/USAGE.md
```

## Convert markdown to mediawiki {#convert_markdown_to_mediawiki}

``` bash
pandoc test.md -f markdown -t mediawiki test.md -o test.wiki
```
## Convert mediawiki to markdown
``` bash
pandoc input.mediawiki -f mediawiki -t markdown -o output.md
```
``` bash
for f in *.mediawiki; do
  pandoc "$f" -f mediawiki -t markdown -o "${f%.mediawiki}.md"
done
```

