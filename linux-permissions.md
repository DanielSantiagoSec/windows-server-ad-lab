# Linux Permissions, User Management, and Log Analysis

This document covers Linux file permission management using chmod and octal notation, the sticky bit, user and group administration, cross-platform log analysis, and port discovery and closure using Nmap.

---

## File Permissions: chmod and Octal Notation

Linux file permissions are represented as three sets of read (r=4), write (w=2), and execute (x=1) bits for owner, group, and others.

**Octal Notation Reference:**

| Octal | Binary | Permissions |
|-------|--------|-------------|
| 7 | 111 | rwx |
| 6 | 110 | rw- |
| 5 | 101 | r-x |
| 4 | 100 | r-- |
| 0 | 000 | --- |

**Examples Used in Lab:**

```bash
# Sensitive config file — owner read/write only, no access for group or others
chmod 600 /etc/app/config.conf

# Script file — owner full, group execute only, others no access
chmod 750 /opt/scripts/backup.sh

# Log directory — owner full, group read/execute, others no access
chmod 750 /var/log/applogs

# Web server content — owner full, group and others read/execute (public files)
chmod 755 /var/www/html

# Verify permissions
ls -la /etc/app/config.conf
```

**Why Octal Matters for Security:**
Permissions like `777` (rwxrwxrwx) are a red flag during any audit — they grant all users full read, write, and execute access. Identifying world-writable files on a system is a standard step in privilege escalation and misconfiguration auditing. Octal notation makes it immediately readable without counting symbolic characters.

---

## Sticky Bit and Least Privilege

**The Sticky Bit:**
The sticky bit, when applied to a directory, prevents users from deleting or renaming files they do not own — even if they have write access to the directory. This is the standard permission on `/tmp`.

```bash
# Apply sticky bit to a shared directory
chmod +t /shared/teamfiles

# Verify — sticky bit shows as 't' in the execute position for others
ls -la /shared/
# drwxrwxrwt 2 root root 4096 Jul 20 12:00 teamfiles
```

**Least Privilege in Practice:**
The sticky bit is a concrete implementation of least privilege at the filesystem level. A user with write access to a shared directory can add files, but cannot delete another user's files. This limits the blast radius of a compromised or malicious account operating in a shared space.

Applied consistently, least privilege means:
- Users have read access where they need to read, write access only where they need to write
- Execute bits are granted only where execution is required
- Sensitive files (configs, keys, logs) are readable only by the owning process or administrator

---

## User and Group Management

**Common Commands Used in Lab:**

```bash
# Add a new user with home directory and bash shell
sudo useradd -m -s /bin/bash username

# Set or change a user's password
sudo passwd username

# Create a new group
sudo groupadd groupname

# Add user to a group (without removing existing group memberships)
sudo usermod -aG groupname username

# View a user's group memberships
groups username
id username

# Lock an account (disable login without deleting)
sudo usermod -L username

# Unlock an account
sudo usermod -U username

# Delete a user (keep home directory for audit purposes)
sudo userdel username

# Delete a user and home directory
sudo userdel -r username
```

**Group-Based Access Control:**
In the lab environment, directory and file access was managed via group ownership rather than individual user ACLs. This mirrors the RBAC approach used in Windows — when a user's role changes, update their group membership and access propagates automatically to all group-controlled resources.

---

## Log Analysis

Log analysis was performed across three platforms during the lab to understand how each records security-relevant events.

### Windows Event Viewer

Key event IDs reviewed during lab:

| Event ID | Description | Security Relevance |
|----------|-------------|-------------------|
| 4624 | Successful logon | Baseline for normal access; anomalous sources warrant investigation |
| 4625 | Failed logon | Repeated failures = potential brute force |
| 4720 | User account created | Track provisioning activity |
| 4726 | User account deleted | Track deprovisioning |
| 4728 | Member added to security group | Track privilege escalation paths |

**Process:** Filtered Security log by Event ID 4624 to review successful logon events. Identified logon type (interactive vs. network), source IP, and logon account. This is the same event used in the Task Scheduler automation task documented in `active-directory-configuration.md`.

### IIS Log Analysis

IIS logs were reviewed to identify web traffic patterns on the lab web server.

- **Log location:** `C:\inetpub\logs\LogFiles\W3SVC1\`
- **Format:** W3C Extended Log Format (space-delimited, timestamped)
- **Fields reviewed:** date, time, c-ip, cs-uri-stem, sc-status, cs(User-Agent)

Identified: repeated 404 errors from a single source IP — consistent with directory enumeration. Reviewed the cs-uri-stem field to identify what paths were being probed.

### Linux Log Analysis

```bash
# Authentication log — successful and failed SSH logins
cat /var/log/auth.log | grep "Failed password"
cat /var/log/auth.log | grep "Accepted password"

# System messages
tail -f /var/log/syslog

# Application-specific logs
ls /var/log/
```

Reviewed `auth.log` for failed SSH login attempts. Found repeated attempts from a single IP against the root account — consistent with automated SSH brute force. Account lockout policy was confirmed as mitigation.

---

## Port Discovery and Closure with Nmap

**Objective:** Identify open ports on a lab host and close unnecessary services.

**Discovery Scan:**

```bash
# SYN scan of lab host
nmap -sS -p- 192.168.10.50

# Output (abbreviated):
# PORT     STATE SERVICE
# 22/tcp   open  ssh
# 80/tcp   open  http
# 3306/tcp open  mysql
# 8080/tcp open  http-proxy
# 9200/tcp open  elasticsearch
```

**Analysis:** Port 3306 (MySQL) and 9200 (Elasticsearch) should not be exposed on the network interface — these services should bind to localhost only. Port 8080 was a dev proxy left running after a testing session.

**Remediation:**

```bash
# Stop unnecessary services
sudo systemctl stop elasticsearch
sudo systemctl disable elasticsearch

# Reconfigure MySQL to bind to localhost only
# In /etc/mysql/mysql.conf.d/mysqld.cnf:
# bind-address = 127.0.0.1
sudo systemctl restart mysql

# Stop dev proxy
sudo kill $(lsof -t -i:8080)

# Verify — rescan
nmap -sS 192.168.10.50
# Only ports 22 and 80 remain open
```

**Lesson:** Open ports are attack surface. Services that don't need to be network-accessible should bind to localhost. A periodic Nmap scan of internal hosts is a lightweight way to catch services that have crept into production exposure. The discipline of scanning your own environment before an attacker does it for you is the same pattern used in the Security Findings Translator project — proactive visibility beats reactive discovery.
