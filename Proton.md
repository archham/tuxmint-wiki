---
title: Proton
description: Infos about Proton.me services
published: true
date: 2026-04-05T08:49:47.857Z
tags: docker, proton, mail
editor: markdown
dateCreated: 2026-04-05T08:42:00.076Z
---

# Proton Mail Bridge
Used to handle applications sending mail, because of enryption of Proton Mail.

Code: https://github.com/dancwilliams/protonmail-bridge-docker

## Docker
We will configure this with only SMTP and no IMAP so we get more security. Only sending is allowed

**Big picture**
``Application (Docker) → SMTP → Proton Bridge (Docker) → Proton Mail``

### Step-by-step setup
```bash
# Clone project & cange into it
git clone https://github.com/dancwilliams/protonmail-bridge-docker.git
cd protonmail-bridge-docker



```