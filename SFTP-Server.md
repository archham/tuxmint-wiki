---
title: SFTP Server on Rocky Linux 9
description: Create a secure SFTP Server on Rocky Linux 9 with best practice
published: true
date: 2026-06-01T12:02:07.672Z
tags: linux, selinux, sftp
editor: markdown
dateCreated: 2026-05-29T09:07:56.454Z
---

# Rocky Linux 9 SFTP Server Best Practice

## Goal

```text
22/tcp    = SFTP only
2222/tcp  = normal SSH shell
```

Design:

```text
sshd.service       -> /etc/ssh/sshd_config       -> port 2222
sshd-sftp.service  -> /etc/ssh/sshd_config_sftp  -> port 22
```

---

# 1. Install Required Packages

```bash
dnf install -y openssh-server firewalld policycoreutils-python-utils
systemctl enable --now firewalld sshd
```

---

# 2. Configure SELinux and Firewall

```bash
semanage port -a -t ssh_port_t -p tcp 2222 2>/dev/null || \
semanage port -m -t ssh_port_t -p tcp 2222

firewall-cmd --permanent --add-service=ssh
firewall-cmd --permanent --add-port=2222/tcp
firewall-cmd --reload
```

---

# 3. Create SFTP Group

```bash
groupadd --system sftpusers 2>/dev/null || true
```

---

# 4. Configure Shell SSH on Port 2222

File: `/etc/ssh/sshd_config`

```sshconfig
Port 2222
Protocol 2
UsePAM yes

Include /etc/ssh/sshd_config.d/*.conf

PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication no
PubkeyAuthentication yes

X11Forwarding no
PermitTunnel no
PermitUserEnvironment no

MaxStartups 10:30:60
MaxSessions 10
MaxAuthTries 4
LoginGraceTime 60
ClientAliveCountMax 0

Subsystem sftp /usr/libexec/openssh/sftp-server
Banner /etc/banner
```

Create drop-in:

File: `/etc/ssh/sshd_config.d/05-sftp-hardening.conf`

```sshconfig
# Block SFTP-only users from shell SSH on port 2222.
DenyGroups sftpusers
```

---

# 5. Configure SFTP-only Service on Port 22

File: `/etc/ssh/sshd_config_sftp`

```sshconfig
Port 22
Protocol 2
UsePAM yes

ListenAddress 0.0.0.0
ListenAddress ::

Subsystem sftp internal-sftp

PermitRootLogin no
PasswordAuthentication yes
KbdInteractiveAuthentication no
PubkeyAuthentication yes

X11Forwarding no
AllowTcpForwarding no
PermitTunnel no
AllowAgentForwarding no
PermitTTY no
PermitUserEnvironment no

AllowGroups sftpusers

Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp -d /upload
    X11Forwarding no
    AllowTcpForwarding no
    PermitTunnel no
    AllowAgentForwarding no
    PermitTTY no
```

---

# 6. Create Dedicated Systemd Service

File: `/etc/systemd/system/sshd-sftp.service`

```ini
[Unit]
Description=OpenSSH SFTP-only service on port 22
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.target
Wants=sshd-keygen.target

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D -f /etc/ssh/sshd_config_sftp $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sshd -t -f /etc/ssh/sshd_config
sshd -t -f /etc/ssh/sshd_config_sftp

systemctl daemon-reload
systemctl restart sshd
systemctl enable --now sshd-sftp.service
```

Verify:

```bash
ss -tlnp | grep sshd
```

Expected:

```text
:22
:2222
```

---

# 7. User Provisioning Script

File: `add-new-sftp-user`

```bash
wget -P /usr/local/sbin/add-new-sftp-user https://raw.githubusercontent.com/archham/linux.scripts/refs/heads/main/sftp/add-new-sftp-user.sh && chmod 700 /usr/local/sbin/add-new-sftp-user
```

---

# 8. Create New Users

Password-based:

```bash
/usr/local/sbin/add_new_sftp_user.sh customer1
passwd customer1
```

SSH-key based:

```bash
/usr/local/sbin/add_new_sftp_user.sh customer1 'ssh-ed25519 AAAA...'
```

---

# 9. Validation

Check shell SSH configuration:

```bash
sshd -T -f /etc/ssh/sshd_config
```

Check SFTP configuration:

```bash
sshd -T -f /etc/ssh/sshd_config_sftp
```

Verify listeners:

```bash
ss -tlnp | grep sshd
```

---

# 10. Testing

```bash
sftp -P 22 customer1@server.example.com
```

Should work.

```bash
ssh -p 2222 customer1@server.example.com
```

Should fail.

```bash
ssh -p 2222 adminuser@server.example.com
```

Should work.

---

# 11. Delete Users

```bash
rm -rf /sftp/DESIRED_USER/
userdel DESIRED_USER
```

# Expected Security Model

| User Type | Port 22 (SFTP) | Port 2222 (SSH) |
|------------|------------|------------|
| Member of `sftpusers` | Allowed | Denied |
| Normal admin user | Denied | Allowed |
| Root | Denied | Denied (`PermitRootLogin no`) |

---

# References

- [Red Hat Enterprise Linux 9 - Securing Networks](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/securing_networks/)
- [Red Hat Enterprise Linux 9 - OpenSSH/SFTP hardening and SSH config behavior](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/securing_networks/)
