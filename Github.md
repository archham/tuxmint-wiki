---
title: Github
description: 
published: true
date: 2026-04-20T13:39:01.972Z
tags: git, github
editor: markdown
dateCreated: 2026-03-16T13:50:59.158Z
---

# Introduction

Here are some usefull infos about Github and how to use it.

# Configurations

``` bash
# Get started: Name, Email, Editor 
git config --global user.name "Example Name"
git config --global user.email "email@example.com"
git config --global core.editor "vim"
```

## Tokens

Go to Settings / Developer Settings / Personal access tokens

Create a Token with the needed permissions.[^1]


## Default text editor

``` bash
git config --global core.editor "vim"
```

or

``` bash
vi ~/.gitconfig

-----------------------
[core]
        editor = vim
-----------------------
```

[^1]: <https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens>