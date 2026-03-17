---
title: Conda
description: 
published: true
date: 2026-03-17T16:45:44.561Z
tags: conda, anaconda, python
editor: markdown
dateCreated: 2026-03-16T13:50:32.086Z
---

## Introduction

Conda is a powerful command line tool for package and environment management that runs on Windows, macOS, and Linux.[^1] 
This page is for general info, how to install and use it.

## Installation

### Linux

#### Prerequisites

- Internet
- Rocky Linux 9[^2]
  - Packages: `git pip libXi libXtst libXrandr libXcursor alsa-lib mesa-libEGL libXcomposite libXScrnSaver libXdamage mesa-libGL`

#### Methods / Possibilities

It is possible to install Conda[^3] directly with a script (recommended)
or manually.

##### Installation witch script (12/2024)

Remember to check for the newest script for the installation![^4]

``` bash
# How to install and use booknlp on Rocky Linux 9

## Resolve dependencies
dnf install git pip libXi libXtst libXrandr libXcursor alsa-lib mesa-libEGL libXcomposite libXScrnSaver libXdamage mesa-libGL -y

## Get script and check checksum
wget https://repo.anaconda.com/archive/Anaconda3-2024.10-1-Linux-x86_64.sh
sha256sum Anaconda3-2024.10-1-Linux-x86_64.sh
* 3ba0a298155c32fbfd80cbc238298560bf69a2df511783054adfc151b76d80d8  Anaconda3-2024.10-1-Linux-x86_64.sh

## Install Conda
bash Anaconda3-2024.10-1-Linux-x86_64.sh
ENTER
q
yes
ENTER
yes

source .bashrc

## Show conda install environment
conda info

## List all installed conda packages
conda list

## Update conda
conda update conda
y
```

## Usage

How to use Conda.

``` bash
# Create virtual environment with fixed interpreter version
conda create -n <NAME_OF_ENV> python=3.9
y

# Activate environment
conda activate <NAME_OF_ENV>

# Deactivate environment
conda deactivate
```

### Example of a project (booknlp)[^5]

``` bash
# Install booknlp
## Still need to debug but works for now

conda create -n booknlp_test python=3.9
y
conda activate booknlp_test

cd /root/anaconda3/envs/booknlp_test

pip install --no-cache-dir "numpy<2"

pip install --no-cache-dir torch==1.7.1 transformers==4.11.3 spacy==3.0.5

python -m spacy download en_core_web_sm

pip install booknlp

git clone https://github.com/booknlp/booknlp.git

cd booknlp/
python setup.py install

cd examples/
python run_booknlp.py
```

## References

[^1]: <https://docs.conda.io/projects/conda/en/latest/user-guide/getting-started.html>
[^2]: <https://www.atlantic.net/vps-hosting/how-to-install-anaconda-python-on-rocky-linux-8/#step-3-install-anaconda>
[^3]: <https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html>
[^4]: <https://repo.anaconda.com/archive/>
[^5]: <https://github.com/booknlp/booknlp>
