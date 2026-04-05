---
title: Proton
description: Infos about Proton.me services
published: true
date: 2026-04-05T08:43:59.363Z
tags: docker, proton, mail
editor: markdown
dateCreated: 2026-04-05T08:42:00.076Z
---

# Proton Mail Bridge
Used to handle applications sending mail, because of enryption of Proton Mail.

Code: https://github.com/archham/protonmail-bridge-docker

## Docker
**Big picture**
``Application (Docker) → SMTP → Proton Bridge (Docker) → Proton Mail``
