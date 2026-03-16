---
title: Ansible
description: 
published: true
date: 2026-03-16T22:22:50.618Z
tags: ansible
editor: markdown
dateCreated: 2026-03-16T13:50:18.538Z
---

![Test](Ansible_logo.svg "Test") ANSIBLE - PLAYBOOK FOR LAMP HAPROXY

<https://www.bogotobogo.com/DevOps/Ansible/Ansible-Playbook-Lamp-HAProxy.php>

In this tutorial, we\'re going to use one of Ansible\'s most complete
example playbooks as a template: lamp_haproxy

<https://www.qnap.com/en/how-to/tutorial/article/how-to-use-reverse-proxy-to-improve-secure-remote-connections>

Manage HAProxy / Nginx / Apache from Web Interface using Roxy-WI

<https://computingforgeeks.com/manage-haproxy-nginx-apache-from-web-interface-using-roxy-wi/?expand_article=1>

Ansible-navigator build your own execution enviroment!

<https://www.dbi-services.com/blog/create-and-manage-ansible-execution-environments/>

## To be sorted {#to_be_sorted}

``` bash
# How to install and run Ansible on one node
## including os-update role

# install Ansible and prepare for use

## ssh into the machine and become root
ssh user@target
sudo su -

## install Ansible with the help of bitbull.ch script
curl -L ansible.bitbull.ch | bash

## add user from whom you would like to operate Ansible
useradd ansible
passwd ansible
usermod -aG wheel ansible

# prepare Ansible user for use

## change into created Ansible user
su - ansible

## generate ssh key pair
ssh-keygen

## edit .bashrc for easier use
echo cd $HOME
vi .bashrc

'''bash
---------
alias cda='cd /etc/ansible'
---------
'''

#  


sadfasdfasdfasdfasdf

## add user from whom you would like to operate Ansible and add to sudoers file
useradd ansible
passwd ansible
usermod -aG wheel ansible
vi /etc/sudoers

'''bash
---------

---------
'''
```

``` bash
ansible-navigator inventory -i inventory1 -m stdout --list
ansible-navigator inventory -i inventory1 -m stdout --graph all
ansible-navigator inventory run file1.yml -m stdout --check #dry run
ansible-navigator inventory run file1.yml -m stdout --syntax-check

Ansible = Modules
Linux OS = Command
sometimes names are identical but written different

yml/yaml both ok

Number of spaces in yml file not defined but must be grater that parent

Example of Playbook
---
- name: VALUE #describes the job
  host: VALUE #defines the affected targets
  vars:
    var1: 100
    var2: 200
  tasks:
    - Module1
        conf/attributes
    - Module2
        conf/attributes

=========

parameter1:
parameter2:
  child parameter:
parameter3:
  
========
  
sample.yml
---
- name: Testing a user module
  hosts: server a
    
ansible-navigator doc -l
  
[Aliases = .bashrc]
 
alias ansdoc=ansible-navigator doc -m stdout
alias ansrun=ansible-navigator rund -m stdout
alias ansinventory=ansible-navigator inventory -m stdout
alias ansdryrun=ansible-navigator --check -m stdout
alias anssyntax=ansible-navigator --syntax-check -m stdout

FQCN = Collection
ansible.builtin.MODULE

collection = Ansammlung von Modulen
    
ansible-navigator doc -l # list
ansible-navigator doc ansible.builtin.group -m stdout # man page of module

ansible.builtin.file
ansible.builtin.copy
ansible.builtin.lineinfile
ansible.builtin.blockfile

ansible.builtin.user
ansible.builtin.group
  
ansible.builtin.dnf
ansible.builtin.apt
ansible.builtin.service

ansible.builtin.get_url
ansible.builtin.uri

ansible.posix.firewalld

ansible-vault create secret_playbook.yaml

ansible-navigator run secret_plybook.yaml -m stdout --vauilt-id @promt
ansible-navigator run secret_plybook.yaml -m stdout --vauilt-id @promt

- name: Describe Playbook
  hosts:
  remote_user: blub
  become: true
  vars:
  vars_files:
    - external_vars:yml
  tasks:
  
ansible.builtin.setup
 --> collect the data/information of the target system

ansible_facts:
  hostname: servera
  FQDN: servera.lab.example.com
  domain: lab.example.com
  interfaces:
    eth0:
      address:
      mtu: 1500
  devices:
    vda:
      partitions:
        vda1:

ansible servera -m setup | less # gather facts

var_1: 100

ansible way of array:
var_2:
  - 100
  - 200
  - 300
  - 400

python way of array:
p_var=[130,120,110,100]

pythoin way of dictionary
p_dict_var={
'user_1':'josef',
'user_2':'joe',
'user_3':'me',
'user_4':'joho',
}


Output gatheriong facts in JSON format
JSON:
   [      ]

ansible way of a dictioonary
users:
  user_1:josef
  user_2:joe
  user_3:me
  user_4:joho

============

make folder on destination server if you need to store variables there
/etc/ansible/facts.d/redhat.fact
it is a ini file type

Loop Structure:

vars:
    var_1:
         - josef
         - joe

tasks:
    - name: Some Task
      Module:
        Attributes:
      loop:
         - Value
         - Value

=====

Conditional Structure

vars:
  var_1: servera

tasks:
  -name: some task
  module:
    Attributes:
  when: var_1 == ansible_facts.hostname


Handlers:

  tasks:
    - name: action
      module:
        Attributes

  handlers:
    - name: Action
      module:
        Attributes:

!!!! Handlers only triggered only when target system has the status changed and the handler musst be notified !!!!!

Hndlers will always be executed at the end!

any failure in the playbook then even the task that changed with the notifyer run the handlers will not be executed! or declare:

force_handlers: true

otherwise some handlers may never be executed
====

block types:

block:
  - Modules
      Attributes:
  - Modules
      Attributes:
  - Modules
      Attributes:

rescue: (if "main blcok" failed)
  - Modules
      Attributes:
  - Modules
      Attributes:
  - Modules
      Attributes:

always: (all modules will run here)
  - Modules
      Attributes:
  - Modules
      Attributes:
  - Modules
      Attributes:

================

Host: Patterns:

- name: Describe Playbook
  hosts: all
  tasks:
    - name: Upgrade software
      dnf:
           name: '*'
           state: latest

hosts: '*'
hosts: all

hosts: jboss
hosts: '*.dns.com'

hosts: ungrouped

hosts: 'server*'

hosts: '172.25.250.*'

hosts: jboss,mysql

hosts: jboss,&mysql #only when common (existing in both groups)

hosts: jboss,!mysql #oposite

hosts: servera,serverd,172.25.250.10,*.redhat.com



ansible-navigator run servera_playbook.yml -m stdout --list-task


==========

Roles:

main.yml

README.md # Info about role

meta/main.yml #write meta related info (creator, etc)

defaults/main.yml #alsow stores variables


ansible-galaxy role init myroles/httpd_server #make a default directory structure

ansible.cfg # declare that you have roles. roles_path=myroles



==========


Collections:

How to install collections with community

https://pkgs.org/i

========

debugging

ansible artifact #ultimate way to debug full deep
log_path=nameoflogfile.log #enable in ansible.cfg

--check --diff
--check
--syntax-check
--steps
--start-at-task
-vvvv

ansible-navigator lint PLAYBOOK

ansible servera -m ping #ping servera via ansible
```
