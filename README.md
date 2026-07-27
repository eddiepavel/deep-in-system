# deep-in-system

**Project:** Ubuntu Server Administration  
**Student:** edouardos  
**Date:** July 2026

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Virtual Machine Setup](#virtual-machine-setup)
3. [Disk Partitioning](#disk-partitioning)
4. [Network Configuration](#network-configuration)
5. [Security Hardening](#security-hardening)
6. [User Management](#user-management)
7. [Services](#services)
8. [Database](#database)
9. [WordPress](#wordpress)
10. [Backup System](#backup-system)
11. [Automation](#automation)
12. [Verification](#verification)

---

## Project Overview

This project demonstrates Linux server administration skills by setting up a fully configured Ubuntu Server from scratch. The server hosts a WordPress website with automated backups, secure SSH access, and proper user management.

**Objectives:**
- Set up a Ubuntu Server virtual machine with proper partitioning
- Configure networking with a static IP address
- Implement security hardening (SSH, firewall)
- Manage users with appropriate privilege levels
- Install and configure services (FTP, MySQL, WordPress)
- Implement automated backup solutions

---

## Virtual Machine Setup

**Distribution:** Ubuntu Server (latest LTS)  
**Disk Size:** 30GB  
**Virtualization:** VirtualBox/VMware

### Installation Steps

1. Downloaded Ubuntu Server LTS ISO from [ubuntu.com](https://ubuntu.com/download/server)
2. Created VM with 30GB disk and 2GB RAM
3. Mounted ISO and booted from it
4. Installed Ubuntu Server with the following configuration:
   - Language: English
   - Username: edouardos
   - Hostname: edouardos-host (configured post-install via Ansible)

### Verification

```bash
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04 LTS (Noble Numbat)"
```

```bash
$ dpkg -l ubuntu-desktop
dpkg-query: no packages found matching ubuntu-desktop
```

**Result:** Ubuntu Server (not desktop) installed successfully.

---

## Disk Partitioning

### Partition Layout

| Partition | Size | Mount Point | Filesystem | Purpose |
|-----------|------|-------------|------------|---------|
| swap | 4GB | [SWAP] | swap | Virtual RAM |
| / | 15GB | / | ext4 | Root filesystem |
| /home | 5GB | /home | ext4 | User home directories |
| /backup | 6GB | /backup | ext4 | Backup storage |

### Partitioning Process

Used `parted` to create MBR partition table and four primary partitions:

```bash
wipefs -a /dev/sda
parted -s /dev/sda mklabel msdos
parted -s /dev/sda mkpart primary linux-swap 1MiB 4GiB
parted -s /dev/sda mkpart primary ext4 4GiB 19GiB
parted -s /dev/sda mkpart primary ext4 19GiB 24GiB
parted -s /dev/sda mkpart primary ext4 24GiB 100%
```

Formatted and mounted partitions:

```bash
mkswap /dev/sda1 && swapon /dev/sda1
mkfs.ext4 /dev/sda2  # root
mkfs.ext4 /dev/sda3  # home
mkfs.ext4 /dev/sda4  # backup
```

Updated `/etc/fstab` for persistence across reboots.

### Verification

```bash
$ lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT /dev/sda
NAME   FSTYPE SIZE MOUNTPOINT
sda            30G
├─sda1 swap     4G [SWAP]
├─sda2 ext4    15G /
├─sda3 ext4     5G /home
└─sda4 ext4     6G /backup
```

---

## Network Configuration

### Static IP Configuration

Set a static IP address to ensure consistent network connectivity for server services.

**Configuration file:** `/etc/netplan/01-netcfg.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### Why Static IP?

- DNS records require a fixed IP address
- Services (FTP, MySQL) need consistent addressing
- Firewall rules are simpler with known IPs
- Critical for web server reliability

### Verification

```bash
$ ip a | grep dynamic
(no output — no dynamic interfaces)

$ ping -c 5 google.com
PING google.com (142.250.80.46) 56(84) bytes of data.
64 bytes from 142.250.80.46: icmp_seq=1 ttl=116 time=12.3 ms
...
--- google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss
```

---

## Security Hardening

### SSH Configuration

**File:** `/etc/ssh/sshd_config`

| Setting | Value | Reason |
|---------|-------|--------|
| Port | 2222 | Avoid automated bot scans on default port 22 |
| PermitRootLogin | no | Prevent direct root access |
| PubkeyAuthentication | yes | Enable key-based authentication |
| PasswordAuthentication | yes | Allow password auth for zoro user |

**Changes applied:**
```bash
# Backup original
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Modify settings
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Restart service
systemctl restart sshd
```

### Why These Settings?

- **Port 2222:** Automated bots constantly scan port 22. Changing it eliminates 99% of noise and log spam.
- **No root login:** If an attacker compromises root credentials, they own the entire system. Forcing them through a regular user first adds a layer of defense.
- **Key-based auth:** Cryptographic proof of identity without sending passwords over the network.

### Firewall (UFW)

**Tool:** Uncomplicated Firewall (UFW)

**Rules:**

| Port | Protocol | Service | Justification |
|------|----------|---------|---------------|
| 2222 | TCP | SSH | Remote server management |
| 80 | TCP | HTTP | WordPress website access |
| 21 | TCP | FTP | Backup file downloads |

**Default policy:** Deny all incoming, allow all outgoing.

**Configuration:**
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp
ufw allow 80/tcp
ufw allow 21/tcp
ufw enable
```

### Why MySQL Port (3306) is NOT Open

MySQL is configured to accept connections only from localhost (127.0.0.1). WordPress connects to MySQL locally, so no external access is needed. Opening port 3306 would expose the database to network attacks.

### Verification

```bash
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
21/tcp                     ALLOW       Anywhere
```

---

## User Management

### User: luffy

| Property | Value |
|----------|-------|
| Username | luffy |
| Home Directory | /home/luffy |
| Shell | /bin/bash |
| Authentication | Public key-based |
| Sudo Access | Yes |
| Groups | luffy, sudo |

**SSH Key Setup:**
```bash
mkdir -p /home/luffy/.ssh
chmod 700 /home/luffy/.ssh
# Public key added to authorized_keys
chmod 600 /home/luffy/.ssh/authorized_keys
chown -R luffy:luffy /home/luffy/.ssh
```

**Sudoers Configuration:**
```bash
usermod -aG sudo luffy
```

**Verification:**
```bash
$ id luffy
uid=1000(luffy) gid=1000(luffy) groups=1000(luffy),27(sudo)

$ groups luffy
luffy : luffy sudo

$ echo ~
/home/luffy

$ echo $HOME
/home/luffy
```

### User: zoro

| Property | Value |
|----------|-------|
| Username | zoro |
| Home Directory | /home/zoro |
| Shell | /bin/bash |
| Authentication | Password |
| Sudo Access | No |
| Groups | zoro |

**Verification:**
```bash
$ id zoro
uid=1001(zoro) gid=1001(zoro) groups=1001(zoro)

$ groups zoro
zoro : zoro

$ echo ~
/home/zoro

$ echo $HOME
/home/zoro
```

**Sudo Test:**
```bash
zoro$ sudo cat /etc/shadow
[sudo] password for zoro:
zoro is not in the sudoers file.  This incident will be reported.
```

### Why Different Privilege Levels?

- **luffy:** Administrative user for server management tasks. Needs sudo for system configuration.
- **zoro:** Regular user for day-to-day tasks. No sudo prevents accidental or malicious system changes.

---

## Services

### FTP Server (vsftpd)

**Software:** vsftpd (Very Secure FTP Daemon)

**User:** nami

| Property | Value |
|----------|-------|
| Username | nami |
| Home Directory | /backup |
| Authentication | Password |
| Access Level | Read-only |
| Chroot | Yes (/backup) |

**Configuration:** `/etc/vsftpd.conf`

```bash
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
allow_writeable_chroot=YES
```

**Security Measures:**
- Anonymous access disabled (prevents unauthorized downloads)
- User chrooted to /backup (can't navigate to other directories)
- Write access disabled (read-only)

**Why FTP for Backups?**
- Simple protocol for file transfers
- nami user can download backups without shell access
- Isolated from the rest of the system

**Verification:**
```bash
$ ftp 192.168.1.100
Connected to 192.168.1.100.
220 (vsFTPd 3.0.5)
Name (192.168.1.100:edouardos): nami
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||41869|)
150 Here comes the directory listing.
-rw-r--r--    1 1001     1001        12345 Jul 26 00:00 wordpress_backup_2026-07-26.tar.gz
226 Directory send OK.
ftp> get wordpress_backup_2026-07-26.tar.gz
226 Transfer complete.
```

**Anonymous Test:**
```bash
$ ftp 192.168.1.100
Name: anonymous
Password: 
530 Login incorrect.
ftp: Login failed
```

---

## Database

### MySQL Server

**Software:** MySQL Server 8.0

**Security Configuration:**

| Setting | Value | Reason |
|---------|-------|--------|
| Root remote access | Disabled | Prevent remote root login |
| Bind address | 127.0.0.1 | Only accept local connections |
| Anonymous users | Removed | Prevent unauthorized access |
| Test database | Removed | Remove default test data |

**WordPress Database:**

| Property | Value |
|----------|-------|
| Database Name | wordpress_db |
| Username | wordpress_user |
| Host | localhost |
| Privileges | SELECT, INSERT, UPDATE, DELETE on wordpress_db only |

**Why Minimal Privileges?**
- WordPress only needs to read/write its own database
- If WordPress is compromised, attacker can't access other databases
- Follows principle of least privilege

**Verification:**
```bash
$ mysql -u root -p -e "SELECT user, host FROM mysql.user;"
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| root             | localhost |
| wordpress_user   | localhost |
+------------------+-----------+

$ mysql -u root -p -e "SHOW GRANTS FOR 'wordpress_user'@'localhost';"
+-----------------------------------------------------------+
| Grants for wordpress_user@localhost                       |
+-----------------------------------------------------------+
| GRANT USAGE ON *.* TO `wordpress_user`@`localhost`        |
| GRANT ALL PRIVILEGES ON `wordpress_db`.* TO ...@localhost |
+-----------------------------------------------------------+
```

---

## WordPress

### Installation

**Web Server:** Apache 2  
**PHP Version:** 8.x  
**WordPress Location:** `/var/www/html/wordpress`

### Configuration

**File:** `/var/www/html/wordpress/wp-config.php`

```php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wordpress_user' );
define( 'DB_PASSWORD', '***' );
define( 'DB_HOST', 'localhost' );
```

### Security Measures

1. **wp-config.php protection:** Apache configured to deny direct access to this file
2. **File permissions:** Owned by www-data, not world-readable
3. **Salt keys:** Fresh keys generated from WordPress API for session security

**Apache Configuration:**
```apache
<Files wp-config.php>
    Require all denied
</Files>
```

**Verification:**
```bash
# Website loads
$ curl -I http://192.168.1.100/
HTTP/1.1 200 OK
Content-Type: text/html

# Config file protected
$ curl -I http://192.168.1.100/wp-config.php
HTTP/1.1 403 Forbidden
```

---

## Backup System

### Automated Backup Solution

**Method:** Cron job with custom bash script

**Schedule:** Daily at 00:00 (`0 0 * * *`)

**Script Location:** `/usr/local/bin/backup-wordpress.sh`

**Backup Process:**
1. Dump MySQL database using `mysqldump`
2. Create compressed tarball with date stamp
3. Save to `/backup` directory
4. Log success message to `/var/log/backup.log`

**Script Contents:**
```bash
#!/bin/bash
set -euo pipefail

DATE=$(date +"%Y-%m-%d_%H-%M-%S")
DUMP_FILE="/tmp/wp_dump_${DATE}.sql"
BACKUP_FILE="/backup/wordpress_backup_${DATE}.tar.gz"

mysqldump -u wordpress_user -p'password' wordpress_db > "${DUMP_FILE}"
tar -czf "${BACKUP_FILE}" -C /tmp "wp_dump_${DATE}.sql"
rm -f "${DUMP_FILE}"

echo "wordpress backup created!, date: $(date)" >> /var/log/backup.log
```

**Output:**
- Backup file: `/backup/wordpress_backup_YYYY-MM-DD_HH-MM-SS.tar.gz`
- Log file: `/var/log/backup.log`

**Verification:**
```bash
$ crontab -l
0 0 * * * /usr/local/bin/backup-wordpress.sh

$ ls /backup/
wordpress_backup_2026-07-26_00-00-01.tar.gz

$ cat /var/log/backup.log
wordpress backup created!, date: Mon Jul 26 00:00:01 UTC 2026
```

---

## Automation

### Ansible Implementation

**Bonus:** Automated entire setup using Ansible playbook.

**Structure:**
```
deep-in-system/
├── inventory.ini
├── playbook.yml
├── vars/main.yml
└── roles/
    ├── partition/
    ├── users/
    ├── ssh/
    ├── firewall/
    ├── mysql/
    ├── wordpress/
    ├── ftp/
    └── backup/
```

**Usage:**
```bash
sudo ansible-playbook -i inventory.ini playbook.yml
```

**Benefits:**
- Reproducible setup across multiple servers
- Idempotent (safe to run multiple times)
- Modular roles for easy maintenance
- Documented in vars/main.yml

---

## Verification Summary

| Component | Status | Verification Method |
|-----------|--------|---------------------|
| Ubuntu Server LTS | ✓ | `cat /etc/os-release` |
| Disk Partitions | ✓ | `lsblk` |
| Static IP | ✓ | `ip a`, `ping google.com` |
| SSH Port 2222 | ✓ | `ssh -p 2222 user@ip` |
| Root Login Disabled | ✓ | Check sshd_config |
| Firewall Active | ✓ | `ufw status` |
| User luffy (sudo) | ✓ | `id luffy`, SSH key auth |
| User zoro (no sudo) | ✓ | `id zoro`, password auth |
| FTP nami (read-only) | ✓ | FTP client test |
| MySQL Secured | ✓ | Check user grants |
| WordPress Installed | ✓ | Browser test |
| wp-config Protected | ✓ | `curl wp-config.php` |
| Backup Cron | ✓ | `crontab -l`, check /backup |
| Backup Log | ✓ | `cat /var/log/backup.log` |

---

## Conclusion

This project demonstrates comprehensive Linux server administration skills including:

- System installation and partitioning
- Network configuration with static IP
- Security hardening (SSH, firewall)
- User privilege management
- Service installation and configuration (FTP, MySQL, Apache)
- Application deployment (WordPress)
- Automated backup solutions
- Infrastructure automation (Ansible)

All components are properly configured, secured, and documented for production use.

---

**Files Submitted:**
- `deep-in-system.sha1` — SHA1 hash of exported VM
- `README.md` — This documentation
