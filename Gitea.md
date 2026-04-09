---
title: Gitea
description: 
published: true
date: 2026-04-09T12:52:45.195Z
tags: linux, gitea, git, selinux, security
editor: markdown
dateCreated: 2026-03-16T13:50:51.959Z
---

The instructions bellow are sourced 100% from [bitbull.ch - Gitea Setup on Rocky10 with SELinux](https://wiki.bitbull.ch/en/gitea) and are here for archival purposes only. All credit to him :-) Visit his site and enjoy the massive knowledge base!

Sourced: 09.04.2026

## My Notes

> **Write down!**
> - MySQL
> Root PW DB=\*\*\*\*
> MARIADB_PW=\*\*\*\*
> - Gitea
> ADMIN_USER=\*\*\*\*
> ADMIN_PW=\*\*\*\*

SQL secure installation

``` bash
genpasswd #used for SQL root account
mysql_secure_installation
enter
n
y
# insert generated pw from before
y
y
y
y
```

Changed Gitea Version
``` bash
GITEA_VERSION=1.24.6
```

Own changes made to app.ini file ``app.ini`` **2025-11-15**
```ini
# Moved vars to top
APP_NAME
RUN_USER
RUN_MODE

# Added vars to [security]
REVERSE_PROXY_LIMIT
REVERSE_PROXY_TRUSTED_PROXIES

# Added [mailer] [migrations] [log] [picture] categories
```

```ini
    cat <<EOF >/etc/gitea/app.ini
    APP_NAME = Gitea: Git with a cup of tea
    RUN_USER = git
    RUN_MODE = prod
    WORK_PATH = /var/lib/gitea/

    [server]
    HTTP_PORT = 3000
    ROOT_URL = https://$HOST
    SSH_DOMAIN = $HOST
    START_SSH_SERVER = true
    SSH_PORT         = 22
    SSH_LISTEN_PORT  = 2222
    OFFLINE_MODE = false
    DOMAIN = $HOST

    [database]
    DB_TYPE = mysql
    HOST = 127.0.0.1:3306
    NAME = gitea
    USER = gitea
    PASSWD = $MARIADB_PW
    SSL_MODE = disable

    [security]
    INSTALL_LOCK = true
    SECRET_KEY = $(genpasswd 32)
    INTERNAL_TOKEN = $(genpasswd 32)
    REVERSE_PROXY_LIMIT = 1
    REVERSE_PROXY_TRUSTED_PROXIES = *

    [lfs]
    PATH = /var/lib/gitea/data/lfs

    [service]
    DISABLE_REGISTRATION = true

    [oauth2]
    JWT_SECRET = $(genpasswd 32)

    [mailer]
    ENABLED = true
    PROTOCOL = smtp
    SMTP_ADDR = gitea.my.domain
    FROM = gitea@my.domain
    USER =
    PASSWD =

    [migrations]
    ALLOWED_DOMAINS = $HOST, github.com, *.github.com

    [log]
    MODE = file
    MAX_DAYS = 32

    [picture]
    AVATAR_MAX_FILE_SIZE = 2097152


    EOF
```
Own changes made to nginx configuration ``gitea.conf`` file **2025-11-19**
```bash
# Added defined upload size limit
client_max_body_size 50M;  # <– increase upload limit
```
```bash
    cat <<EOF >/etc/nginx/conf.d/gitea.conf
    server {
        listen 443 ssl;
        server_name $HOST;

        client_max_body_size 50M;

        ssl_certificate /etc/pki/tls/certs/gitea.crt;
        ssl_certificate_key /etc/pki/tls/private/gitea.key;
        ssl_dhparam /etc/nginx/dhparam.pem;
        ssl_protocols TLSv1.2;
        add_header Strict-Transport-Security "max-age=31536000; 
        includeSubDomains; preload" always;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
    EOF
```
# Setup
- OS: Rocky Linux 10 minimal
- FQDN: gitea01.domain.tld

## Disable SELinux

For now, we set selinux to permissive, once all rules are set and we have no violations anymore, we set to enforce again
```bash
    setenforce 0
    sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
```
## VARS and Functions
```bash
    HOST=$(hostname -f)

    genpasswd() {
        local l=$1
            [ "$l" == "" ] && l=20
            tr -dc A-Za-z0-9_ < /dev/urandom | head -c ${l} | xargs
    }
```
## Install software
install reverse proxy and base software
```bash
dnf -y install openssl wget nginx git policycoreutils-python-utils
```
## Move system sshd port to 222
```bash
semanage port -a -t ssh_port_t -p tcp 222
sed -i 's/^#Port 22/Port 222/' /etc/ssh/sshd_config
systemctl restart sshd
```
## Configure Firewall
```bash
# open https
firewall-cmd --permanent --add-port=443/tcp
# open new sshd port
firewall-cmd --permanent --add-port=222/tcp
# forward port 22 to gitea
firewall-cmd --permanent --add-forward-port=port=22:proto=tcp:toport=2222
# load new rules
firewall-cmd --reload
```
## Install/Configure Mariadb
- install and enable
```bash
dnf install -y mariadb-server
systemctl enable --now mariadb
```
- secure mariadb
```bash
mysql_secure_installation
```
- create gitea database user
```bash
MARIADB_PW=$(genpasswd)
echo MARIADB_PW=$MARIADB_PW
mysql -u root -p <<EOF
CREATE DATABASE gitea CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
CREATE USER 'gitea'@'localhost' IDENTIFIED BY '$MARIADB_PW';
GRANT ALL PRIVILEGES ON gitea.* TO 'gitea'@'localhost';
FLUSH PRIVILEGES;
EOF
```
## Install Gitea
- create gitea user
```bash
useradd --system --shell /usr/bin/false --comment 'Git Version Control' --create-home --home-dir /var/lib/gitea git
mkdir -p /var/lib/gitea/data
mkdir -p /etc/gitea
chown -R git:git /var/lib/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```
- Install gitea
```bash
    GITEA_VERSION=1.23.8
    wget -O /usr/local/bin/gitea https://dl.gitea.com/gitea/$GITEA_VERSION/gitea-$GITEA_VERSION-linux-amd64
    chown root:git /usr/local/bin/gitea
    chmod 750 /usr/local/bin/gitea
```
- create systemd service
```bash
cat <<EOF >/etc/systemd/system/gitea.service
[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target
Requires=mariadb.service

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/var/lib/gitea GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOF
```
- create gitea config
```bash
cat <<EOF >/etc/gitea/app.ini
WORK_PATH = /var/lib/gitea/

[server]
APP_NAME = Gitea: Git with a cup of tea
RUN_USER = git
RUN_MODE = prod
HTTP_PORT = 3000
ROOT_URL = https://$HOST
SSH_DOMAIN = $HOST
START_SSH_SERVER = true
SSH_PORT         = 22
SSH_LISTEN_PORT  = 2222
OFFLINE_MODE = false

[database]
DB_TYPE = mysql
HOST = 127.0.0.1:3306
NAME = gitea
USER = gitea
PASSWD = $MARIADB_PW
SSL_MODE = disable

[security]
INSTALL_LOCK = true
SECRET_KEY = $(genpasswd 32)
INTERNAL_TOKEN = $(genpasswd 32)

[lfs]
PATH = /var/lib/gitea/data/lfs

[service]
DISABLE_REGISTRATION = true

[oauth2]
JWT_SECRET = $(genpasswd 32)
EOF
```
```bash
chown root:git /etc/gitea/app.ini
chmod 660 /etc/gitea/app.ini

systemctl daemon-reload
systemctl enable --now gitea
```
## nginx configuration
```bash
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout /etc/pki/tls/private/gitea.key \
  -out /etc/pki/tls/certs/gitea.crt \
  -days 3650 -subj "/CN=$HOST"
```
`chown root:root /etc/pki/tls/private/gitea.key`\
`chmod 600 /etc/pki/tls/private/gitea.key`

`openssl dhparam -out /etc/nginx/dhparam.pem 4096`

    cat <<EOF >/etc/nginx/conf.d/gitea.conf
    server {
        listen 443 ssl;
        server_name $HOST;

        ssl_certificate /etc/pki/tls/certs/gitea.crt;
        ssl_certificate_key /etc/pki/tls/private/gitea.key;
        ssl_dhparam /etc/nginx/dhparam.pem;
        ssl_protocols TLSv1.2;
        add_header Strict-Transport-Security "max-age=31536000; 
        includeSubDomains; preload" always;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;

        location / {
            proxy_pass http://127.0.0.1:3000;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
    EOF

`systemctl enable --now nginx`

## create gitea application admin

Change to git user, which has no sell as default

`su -s /bin/bash git`

    genpasswd() {
      local l=$1
            [ "$l" == "" ] && l=20
            tr -dc A-Za-z0-9_ < /dev/urandom | head -c ${l} | xargs
     }

    ADMIN_USER=$(genpasswd 8)
    ADMIN_PW=$(genpasswd)

    echo ADMIN_USER=$ADMIN_USER
    echo ADMIN_PW=$ADMIN_PW
    gitea admin user create --admin --username $ADMIN_USER --password "$ADMIN_PW" --email admin@domain.tld --config /etc/gitea/app.ini
    exit

# SELinux configuration

## Install selinux tools as root

`dnf install -y policycoreutils-devel checkpolicy setroubleshoot-server setools-console`\
`> /var/log/audit/audit.log`\
`touch /.autorelabel`\
`reboot`\
`ssh -lroot $HOST -p222`

`mkdir -p /root/selinux/gitea`\
`cd /root/selinux/gitea`

[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}

## Generate policy default ruleset

`for r in appstream baseos extras ; do dnf config-manager --set-disabled $r; done`\
`sepolicy generate --init /usr/local/bin/gitea`\
`for r in appstream baseos extras ; do dnf config-manager --set-enabled $r; done`

## Modify ruleset defaults

Rename gitea_var_lib_t

`grep gitea_var_lib_t gitea*`\
`sed -i 's/gitea_var_lib_t/gitea_data_t/g' gitea*`

- Hide permissive setting, we want to use it in enforcing mode

`sed -i 's/^permissive/#permissive/' gitea.te`

- Add file context rule

`echo '/etc/gitea(/.*)?                gen_context(system_u:object_r:gitea_conf_t,s0)' >> gitea.fc`

- `vim gitea.te`

<!-- -->

    # add in declaration
    type gitea_conf_t;
    files_type(gitea_conf_t)

    type gitea_port_t;
    corenet_port(gitea_port_t)

- disable rpm building cmd

`sed -i 's/^rpmbuild/#rpmbuild/' gitea.sh`

- build and load new rules

`bash gitea.sh`

[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}

## Apply new context lables

- Label ports

`semanage port -a -t gitea_port_t -p tcp 2222`\
`semanage port -a -t gitea_port_t -p tcp 3000`

- Restart gitea

`systemctl restart gitea`

- Verify domain

<!-- -->

     ps -ef -Z | grep gitea
       system_u:system_r:gitea_t:s0    git         1714       1 10 07:41 ?        00:00:01 /usr/local/bin/gitea web --config /etc/gitea/app.ini

- verify port lables

<!-- -->

    semanage port --list | grep gitea
       gitea_port_t                   tcp      2222, 3000

- verify file lables

<!-- -->

    ls -lZ /etc/gitea/app.ini /var/lib/gitea/data/
       -rw-rw----.  1 root git system_u:object_r:gitea_conf_t:s0  681 Jul 11 07:23 /etc/gitea/app.ini

       /var/lib/gitea/data/:
       drwxr-xr-x. 2 git git system_u:object_r:gitea_data_t:s0  6 Jul 11 07:23 actions_artifacts
       [...]

[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}

## Define additional SELinux rules

- Reset alerts and verify alerts

`> /var/log/audit/audit.log`\
`systemctl restart gitea`

- Now test the application with all its functions

<!-- -->

- verify new alerts

`sealert -a /var/log/audit/audit.log `\
`grep 'run: sealert' /var/log/messages `\
`ausearch  --raw | audit2allow`

- Review, understand and then add this to gitea.te, gitea.fc, gitea.if
- Then rebuild/install policy with \`bash gitea.sh\`

<!-- -->

- finally start over until you have no violations anymore

\> /var/log/audit/audit.log reboot sealert -a /var/log/audit/audit.log
grep \'run: sealert\' /var/log/messages ausearch \--raw \| audit2allow
ausearch -c \'git\' \--raw \| audit2allow -R

# ReEnable SELinux

Now we switch back to enforcing and test again

`setenforce 1`\
`sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config`\
`grubby --update-kernel ALL --remove-args selinux`\
`touch /.autorelabel`\
`reboot`\
`sealert -a /var/log/audit/audit.log `

## Test if rules are working as expected

`chcon -t home_root_t /etc/gitea/app.ini`\
`> /var/log/audit/audit.log`\
`reboot`\
`sealert -a /var/log/audit/audit.log`\
`restorecon -FRv /etc/gitea/`\
`> /var/log/audit/audit.log`\
`reboot`\
`sealert -a /var/log/audit/audit.log`

    semanage port -a -t ssh_port_t -p tcp 2222
    semanage port -a -t ssh_port_t -p tcp 3000
    > /var/log/audit/audit.log                
    reboot                                    
    sealert -a /var/log/audit/audit.log 
    ss -tan # port 2222 and 3000 are not listening!
    semanage port -a -t gitea_port_t -p tcp 2222
    semanage port -a -t gitea_port_t -p tcp 3000
    semanage port -l | grep gitea_
       gitea_port_t                   tcp      2222, 3000
    > /var/log/audit/audit.log
    reboot
    sealert -a /var/log/audit/audit.log
    ss -tan # port 2222 and 3000 are NOW listening!

[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}

# Appendix

## Final policy files

</pre>
<div class="toccolours mw-collapsible mw-collapsed" style="width:60%">

File: `<b>`{=html}gitea.fc`</b>`{=html} Modified: `<b>`{=html}2025-07-12
12:41:42`</b>`{=html}

<div class="mw-collapsible-content">

    # Set the security context for the Gitea executable
    /usr/local/bin/gitea        --  gen_context(system_u:object_r:gitea_exec_t,s0)
    # Set the security context for Gitea data directories
    /var/lib/gitea(/.*)?        gen_context(system_u:object_r:gitea_data_t,s0)
    # Set the security context for Gitea configuration files
    /etc/gitea(/.*)?        gen_context(system_u:object_r:gitea_conf_t,s0)

</div>
</div>
<div class="toccolours mw-collapsible mw-collapsed" style="width:60%">

File: `<b>`{=html}gitea.if`</b>`{=html} Modified: `<b>`{=html}2025-07-12
12:43:07`</b>`{=html}

<div class="mw-collapsible-content">

    # Interface to allow domain transition to gitea_t when executing gitea_exec_t
    interface(`gitea_domtrans',`
        gen_require(`
            type gitea_t, gitea_exec_t;
        ')

        corecmd_search_bin($1)
        domtrans_pattern($1, gitea_exec_t, gitea_t)
    ')

    # Interface to allow execution of gitea_exec_t in the caller's domain
    interface(`gitea_exec',`
        gen_require(`
            type gitea_exec_t;
        ')

        corecmd_search_bin($1)
        can_exec($1, gitea_exec_t)
    ')

    # Interface to allow searching of Gitea library directories
    interface(`gitea_search_lib',`
        gen_require(`
            type gitea_data_t;
        ')

        allow $1 gitea_data_t:dir search_dir_perms;
        files_search_var_lib($1)
    ')

    # Interface to allow reading of Gitea library files
    interface(`gitea_read_lib_files',`
        gen_require(`
            type gitea_data_t;
        ')

        files_search_var_lib($1)
        read_files_pattern($1, gitea_data_t, gitea_data_t)
    ')

    # Interface to allow management of Gitea library files
    interface(`gitea_manage_lib_files',`
        gen_require(`
            type gitea_data_t;
        ')

        files_search_var_lib($1)
        manage_files_pattern($1, gitea_data_t, gitea_data_t)
    ')

    # Interface to allow management of Gitea library directories
    interface(`gitea_manage_lib_dirs',`
        gen_require(`
            type gitea_data_t;
        ')

        files_search_var_lib($1)
        manage_dirs_pattern($1, gitea_data_t, gitea_data_t)
    ')


    # Interface to allow administration of a Gitea environment
    interface(`gitea_admin',`
        gen_require(`
            type gitea_t;
            type gitea_data_t;
        ')

        allow $1 gitea_t:process { signal_perms };
        ps_process_pattern($1, gitea_t)

        tunable_policy(`deny_ptrace',`',`
            allow $1 gitea_t:process ptrace;
        ')

        files_search_var_lib($1)
        admin_pattern($1, gitea_data_t)
        optional_policy(`
            systemd_passwd_agent_exec($1)
            systemd_read_fifo_file_passwd_run($1)
        ')
    ')

</div>
</div>
<div class="toccolours mw-collapsible mw-collapsed" style="width:60%">

File: `<b>`{=html}gitea.te`</b>`{=html} Modified: `<b>`{=html}2025-07-12
12:44:36`</b>`{=html}

<div class="mw-collapsible-content">

    # Define the Gitea SELinux policy module version
    policy_module(gitea, 1.0.0)

    ########################################
    # Required types and classes for Gitea SELinux policy
    require {
      type gitea_t;
      type gitea_data_t;
      type gitea_port_t;                      
      type gitea_conf_t;                               
      class dir search;
      class process setpgid;
      class file { getattr map open read execute execute_no_trans};
      class tcp_socket { accept bind listen name_bind name_connect connect create getattr getopt setopt read write};
      class udp_socket { connect create getattr setopt };
      type tmp_t;
      type httpd_t;
    } 

    type gitea_t;
    type gitea_exec_t;
    # Initialize the Gitea daemon domain
    init_daemon_domain(gitea_t, gitea_exec_t)

    # permissive gitea_t;

    type gitea_data_t;
    files_type(gitea_data_t)

    type gitea_conf_t;
    files_type(gitea_conf_t)

    type gitea_port_t;
    corenet_port(gitea_port_t)

    ########################################
    # gitea policy definitions #############
    # Allow Gitea to read and write to its own FIFO files
    allow gitea_t self:fifo_file rw_fifo_file_perms;
    # Allow Gitea to create and use Unix stream sockets
    allow gitea_t self:unix_stream_socket create_stream_socket_perms;

    # Allow Gitea to search its configuration directory
    allow gitea_t gitea_conf_t:dir search;             
    # Allow Gitea to get attributes, open, and read its configuration files
    allow gitea_t gitea_conf_t:file { getattr open read };

    # Allow HTTPD to connect to Gitea's TCP socket
    allow httpd_t gitea_port_t:tcp_socket name_connect;

    # Allow Gitea to manage its data directories
    manage_dirs_pattern(gitea_t, gitea_data_t, gitea_data_t)
    # Allow Gitea to manage its data files
    manage_files_pattern(gitea_t, gitea_data_t, gitea_data_t)
    # Allow Gitea to manage its symbolic link files
    manage_lnk_files_pattern(gitea_t, gitea_data_t, gitea_data_t)
    # Allow file transitions for Gitea in /var/lib
    files_var_lib_filetrans(gitea_t, gitea_data_t, { dir file lnk_file })

    # Allow Gitea to use interactive file descriptors
    domain_use_interactive_fds(gitea_t)

    # Allow Gitea to read files in /etc
    files_read_etc_files(gitea_t)

    # Allow Gitea to read localization files
    miscfiles_read_localization(gitea_t)

    # Define file transition patterns for Gitea
    filetrans_pattern(gitea_t, gitea_data_t, gitea_data_t, file);
    filetrans_pattern(gitea_t, gitea_data_t, gitea_data_t, dir);
    filetrans_pattern(gitea_t, tmp_t, gitea_data_t, { file dir });

    # Allow Gitea to map its data files
    allow gitea_t gitea_data_t:file map;
    # Allow Gitea to bind to its TCP socket
    allow gitea_t gitea_port_t:tcp_socket name_bind;
    # Allow Gitea to set process group ID
    allow gitea_t self:process setpgid;
    # Allow Gitea to perform operations on its TCP socket
    allow gitea_t self:tcp_socket { accept bind listen read write};
    # Allow Gitea to read the password file
    auth_read_passwd_file(gitea_t)
    # Allow Gitea to execute shell commands
    corecmd_check_exec_shell(gitea_t)
    # Allow Gitea to execute binaries
    corecmd_exec_bin(gitea_t)
    # Allow Gitea to bind to generic TCP nodes
    corenet_tcp_bind_generic_node(gitea_t)
    # Allow Gitea to read from sysfs
    dev_read_sysfs(gitea_t)
    # Allow Gitea to read network sysctl settings
    kernel_read_net_sysctls(gitea_t)
    # Allow Gitea to search network sysctl settings
    kernel_search_network_sysctl(gitea_t)

    # Allow Gitea to execute its own executable without transition
    allow gitea_t gitea_exec_t:file execute_no_trans;
    # Allow Gitea to execute its data files
    allow gitea_t gitea_data_t:file execute;
    # Allow Gitea to execute its data files without transition
    allow gitea_t gitea_data_t:file execute_no_trans;
    # Allow Gitea to connect to its TCP socket
    allow gitea_t gitea_port_t:tcp_socket name_connect;  
    # Allow Gitea to perform various operations on its TCP socket
    allow gitea_t self:tcp_socket { connect create getattr getopt setopt };
    # Allow Gitea to perform various operations on its UDP socket
    allow gitea_t self:udp_socket { connect create getattr setopt };
    # Allow Gitea to connect to HTTP ports
    corenet_tcp_connect_http_port(gitea_t);
    # Allow Gitea to connect to MySQL ports
    corenet_tcp_connect_mysqld_port(gitea_t)
    # Allow Gitea to read generic certificates
    miscfiles_read_generic_certs(gitea_t);
    # Allow Gitea to read system network configuration
    sysnet_read_config(gitea_t);

</div>
</div>

## Boolean Analysis: domain_can_mmap_files

- Lets see what domain_can_mmap_files contains

<!-- -->

    getsebool -a | grep domain_can_mmap_files
      domain_can_mmap_files --> off


    semanage boolean -l | grep domain_can_mmap_files
       domain_can_mmap_files          (off  ,  off)  Allow any process to mmap any file on system with attribute file_type.

    sesearch -b domain_can_mmap_files -A
       allow domain file_type:blk_file map; [ domain_can_mmap_files ]:True
       allow domain file_type:chr_file map; [ domain_can_mmap_files ]:True
       allow domain file_type:file map; [ domain_can_mmap_files ]:True
       allow domain file_type:lnk_file map; [ domain_can_mmap_files ]:True

[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}
[index.php?title=Category:Linux](index.php?title=Category:Linux "index.php?title=Category:Linux"){.wikilink}
[index.php?title=Category:SELinux](index.php?title=Category:SELinux "index.php?title=Category:SELinux"){.wikilink}
[index.php?title=Category:Security](index.php?title=Category:Security "index.php?title=Category:Security"){.wikilink}
[index.php?title=Category:Gita](index.php?title=Category:Gita "index.php?title=Category:Gita"){.wikilink}
[index.php?title=Category:Git](index.php?title=Category:Git "index.php?title=Category:Git"){.wikilink}
