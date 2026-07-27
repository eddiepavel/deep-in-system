# STUDY GUIDE — deep-in-system Audit Prep

**Your name:** edouardos  
**Audit date:** TBD

---

## Table of Contents

1. [Quick Reference](#quick-reference)
2. [Concepts You Must Explain](#concepts-you-must-explain)
3. [Commands You Must Know](#commands-you-must-know)
4. [Verification Checklist](#verification-checklist)
5. [The Kratos Exam](#the-kratos-exam)
6. [Common Audit Questions](#common-audit-questions)
7. [Troubleshooting](#troubleshooting)

---

## Quick Reference

| Item | Value |
|------|-------|
| Your username | edouardos |
| Hostname | edouardos-host |
| SSH port | 2222 |
| Luffy auth | SSH key (no password) |
| Zoro auth | Password (no sudo) |
| Nami | FTP user, read-only /backup |
| WordPress | http://VM_IP/ |
| Backup cron | 0 0 * * * (daily at midnight) |
| Backup log | /var/log/backup.log |

---

## Concepts You Must Explain

The auditor will ask you to explain these. Practice saying them out loud.

### 1. What is sudo?

**Short answer:** Sudo lets a regular user run specific commands with root privileges. The user must type their password, and the action is logged.

**Key points:**
- "Super User DO" — executes commands as another user (usually root)
- Provides fine-grained access control, not full root shell
- Logs every command to `/var/log/auth.log`
- Safer than sharing root password
- Configured in `/etc/sudoers` file

**Why it's better than root:**
- Root: Everything runs with full power, no audit trail
- Sudo: Specific commands only, logged, requires password

---

### 2. What is SSH?

**Short answer:** Secure Shell — encrypted protocol for remote command-line access to a server.

**Key points:**
- Replaces insecure protocols (telnet, rlogin)
- Encrypts all traffic between client and server
- Uses public-key cryptography for authentication
- Default port: 22 (we changed to 2222)
- Server runs `sshd` daemon

**Two authentication methods:**
1. Password: User types password (brute-forceable)
2. Public key: Client has private key, server has public key (math proves identity)

---

### 3. What is a netmask?

**Short answer:** Defines which part of an IP address is the network and which is the host.

**Example:**
- IP: 192.168.1.100
- Netmask: 255.255.255.0 (/24)
- Network: 192.168.1.0 (first 24 bits)
- Host: .100 (last 8 bits)

**Why it matters:** Determines how many devices can be on the same network. /24 = 254 hosts.

---

### 4. Why static IP for a web server?

**Short answer:** DNS records point to one IP. If it changes, nobody can reach your server.

**Key points:**
- DNS A records map domain → IP address
- Dynamic IP changes on reboot (DHCP)
- Static IP is permanent
- Services need consistent addressing
- Firewall rules reference specific IPs

---

### 5. What is a firewall?

**Short answer:** Filters network traffic based on rules. Blocks unauthorized access, allows legitimate traffic.

**Key points:**
- Default policy: deny all, then open specific ports
- We use UFW (Uncomplicated Firewall)
- Open ports: 2222 (SSH), 80 (HTTP), 21 (FTP)
- MySQL port (3306) NOT open — only local connections needed
- Logs blocked attempts for security review

---

### 6. What is SSH server?

**Short answer:** Daemon (background service) that listens for encrypted remote connections.

**Key points:**
- Runs as `sshd` service
- Listens on port 2222 (we changed from 22)
- Handles authentication (key or password)
- Creates encrypted tunnel for commands
- Config file: `/etc/ssh/sshd_config`

---

### 7. What is FTP?

**Short answer:** File Transfer Protocol — standard protocol for transferring files over a network.

**Key points:**
- We use vsftpd (Very Secure FTP Daemon)
- User nami connects to download backups
- Chrooted to /backup — can't see other directories
- Anonymous access disabled for security
- Read-only — nami cannot modify files

---

### 8. What is a cronjob?

**Short answer:** Time-based task scheduler in Unix/Linux. Runs commands at specified intervals.

**Syntax:** `minute hour day month weekday command`

**Examples:**
- `0 0 * * *` — daily at midnight
- `* * * * *` — every minute
- `0 2 * * 0` — every Sunday at 2 AM

**Our backup job:** `0 0 * * * /usr/local/bin/backup-wordpress.sh`

---

### 9. Why backups are important?

**Short answer:** Protects against data loss from hardware failure, human error, attacks, or disasters.

**Key points:**
- Hardware fails — disks crash, power outages
- Humans make mistakes — delete wrong files
- Security incidents — ransomware encrypts data
- Natural disasters — fire, flood
- Backups let you restore to a known good state
- Regular backups minimize data loss window

---

### 10. What is a database?

**Short answer:** Organized collection of data, structured for efficient retrieval and modification.

**Key points:**
- We use MySQL — relational database
- WordPress stores all data in MySQL (posts, users, settings)
- Tables, rows, columns structure
- SQL (Structured Query Language) for queries
- User `wordpress_user` has limited privileges (only wordpress_db)

---

## Commands You Must Know

### System Information

```bash
# OS version
cat /etc/os-release

# Hostname
hostname

# Current user
whoami
id
groups

# Disk partitions
lsblk
df -h

# Network
ip a
ip route
ping -c 5 google.com
```

### User Management

```bash
# Create user
sudo useradd -m -s /bin/bash username

# Set password
sudo passwd username

# Add to sudo group
sudo usermod -aG sudo username

# Check user info
id username
groups username
cat /etc/passwd | grep username
```

### SSH

```bash
# Test SSH connection
ssh -p 2222 username@VM_IP

# Generate SSH key
ssh-keygen -t rsa -b 4096

# Check SSH config
cat /etc/ssh/sshd_config | grep -E "^(Port|PermitRootLogin)"

# Restart SSH
sudo systemctl restart sshd
```

### Firewall

```bash
# Check status
sudo ufw status verbose

# Add rule
sudo ufw allow 2222/tcp

# Delete rule
sudo ufw delete allow 2222/tcp

# Enable/disable
sudo ufw enable
sudo ufw disable
```

### MySQL

```bash
# Login
mysql -u root -p

# Show databases
SHOW DATABASES;

# Show users
SELECT user, host FROM mysql.user;

# Show grants
SHOW GRANTS FOR 'wordpress_user'@'localhost';

# Test connection
mysql -u wordpress_user -p -e "USE wordpress_db; SELECT 1;"
```

### FTP

```bash
# Test connection
ftp VM_IP

# Login as nami
# Enter username: nami
# Enter password: your_password

# List files
ls

# Download file
get filename

# Exit
bye
```

### Backup

```bash
# Check cron jobs
crontab -l

# Check backup files
ls -la /backup/

# Check backup log
cat /var/log/backup.log

# Run backup manually
sudo /usr/local/bin/backup-wordpress.sh
```

---

## Verification Checklist

Go through this list before the audit. Every item must pass.

### VM Setup
- [ ] Ubuntu Server LTS installed (not desktop)
- [ ] 30GB disk
- [ ] Partitions: swap 4G, / 15G, /home 5G, /backup 6G
- [ ] Hostname: edouardos-host

### Network
- [ ] Static IP configured
- [ ] No dynamic interfaces (`ip a | grep dynamic` returns nothing)
- [ ] Internet works (`ping -c 5 google.com`)

### SSH
- [ ] Port 2222 (`cat /etc/ssh/sshd_config | grep "^Port"`)
- [ ] Root login disabled (`cat /etc/ssh/sshd_config | grep "PermitRootLogin"`)

### Users
- [ ] luffy exists, in sudo group, home /home/luffy
- [ ] luffy SSH with key (no password)
- [ ] zoro exists, NOT in sudo group, home /home/zoro
- [ ] zoro SSH with password
- [ ] zoro cannot sudo (`sudo cat /etc/shadow` fails)

### Firewall
- [ ] UFW active (`sudo ufw status`)
- [ ] Only ports 2222, 80, 21 open
- [ ] MySQL port 3306 NOT open

### MySQL
- [ ] Root remote access disabled
- [ ] WordPress user exists with limited privileges
- [ ] Only accepts localhost connections

### WordPress
- [ ] Website loads in browser
- [ ] wp-config.php not accessible (403 Forbidden)
- [ ] Can login to WordPress admin

### FTP
- [ ] nami can login
- [ ] nami can list /backup files
- [ ] nami can download files
- [ ] Anonymous login fails

### Backup
- [ ] Cron job exists (`crontab -l`)
- [ ] Backup files appear in /backup after cron runs
- [ ] /var/log/backup.log exists and has entries

---

## The Kratos Exam

**Time limit:** 10 minutes  
**You must do this live, in front of the auditor.**

### Step-by-Step

```bash
# 1. Generate SSH key pair (on the VM)
ssh-keygen -t rsa -b 4096 -f /tmp/kratos_key -N ""

# 2. Create user
sudo useradd -m -s /bin/bash kratos

# 3. Set up SSH directory
sudo mkdir -p /home/kratos/.ssh
sudo cp /tmp/kratos_key.pub /home/kratos/.ssh/authorized_keys
sudo chown -R kratos:kratos /home/kratos/.ssh
sudo chmod 700 /home/kratos/.ssh
sudo chmod 600 /home/kratos/.ssh/authorized_keys

# 4. Add to sudo group
sudo usermod -aG sudo kratos

# 5. Test SSH connection
ssh -i /tmp/kratos_key kratos@localhost

# 6. Test sudo (as kratos)
sudo whoami
# Should output: root

# 7. Exit
exit
```

### Practice Until You Can Do It In Under 5 Minutes

**Common mistakes:**
- Forgetting `sudo` on mkdir
- Wrong permissions on .ssh (must be 700) or authorized_keys (must be 600)
- Forgetting to `chown` the .ssh directory
- Using wrong user (kratos, not anything else)

---

## Common Audit Questions

### Q: Can you explain what is sudo group in Linux?

**A:** The sudo group contains users who can execute commands with root privileges using the `sudo` command. When a user is in this group, they can run administrative tasks without needing the root password. Every sudo command is logged for audit purposes.

---

### Q: Can you explain the SSH configuration?

**A:** We changed the default port from 22 to 2222 to avoid automated bot scans. We disabled root login so attackers can't directly target the root account. We enabled both key-based and password authentication for different user needs.

---

### Q: Why is the MySQL port not open in the firewall?

**A:** WordPress runs on the same server as MySQL, so it connects locally (localhost/127.0.0.1). There's no need for external access. Opening port 3306 would expose the database to network attacks and brute-force attempts.

---

### Q: Can you explain the backup system?

**A:** A cron job runs every day at midnight. It dumps the WordPress database using mysqldump, compresses it into a tarball with the date in the filename, saves it to /backup, and logs the success message to /var/log/backup.log. The backup files are accessible via FTP by the nami user.

---

### Q: Why is the config file not public accessible?

**A:** wp-config.php contains database credentials (username and password). If someone could read this file, they would have full access to the WordPress database. Apache is configured to deny direct access to this file while WordPress can still read it.

---

### Q: What happens if zoro tries to sudo?

**A:** zoro is not in the sudo group, so the command fails immediately with "zoro is not in the sudoers file. This incident will be reported." The attempt is logged in /var/log/auth.log.

---

### Q: How does the FTP chroot work?

**A:** When nami connects via FTP, the chroot jail makes /backup appear as the root directory. The user cannot navigate above /backup or access any other part of the filesystem. This isolates the backup data from the rest of the system.

---

## Troubleshooting

### SSH won't connect on port 2222

```bash
# Check if sshd is running
sudo systemctl status sshd

# Check if port is open
ss -tlnp | grep 2222

# Check firewall
sudo ufw status | grep 2222

# Check sshd config
cat /etc/ssh/sshd_config | grep "^Port"
```

### WordPress shows blank page

```bash
# Check Apache status
sudo systemctl status apache2

# Check PHP errors
sudo tail -f /var/log/apache2/error.log

# Check file permissions
ls -la /var/www/html/wordpress/
```

### MySQL connection refused

```bash
# Check MySQL status
sudo systemctl status mysql

# Check if listening on localhost
ss -tlnp | grep 3306

# Test connection
mysql -u wordpress_user -p -e "SELECT 1;"
```

### FTP login fails

```bash
# Check vsftpd status
sudo systemctl status vsftpd

# Check if listening
ss -tlnp | grep :21

# Check user exists
id nami

# Check vsftpd config
cat /etc/vsftpd.conf | grep -E "(anonymous|local|chroot)"
```

### Backup not running

```bash
# Check cron
crontab -l

# Check if script is executable
ls -la /usr/local/bin/backup-wordpress.sh

# Run manually
sudo /usr/local/bin/backup-wordpress.sh

# Check log
cat /var/log/backup.log
```

---

## Final Tips

1. **Practice explaining concepts out loud** — not just reading them
2. **Time yourself on the kratos exam** — get under 5 minutes
3. **Know your passwords** — write them down temporarily if needed
4. **Stay calm** — if something fails, troubleshoot step by step
5. **Understand, don't memorize** — the auditor can tell the difference

---

**Good luck! You've got this.**
