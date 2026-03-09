# Linux User Management & Scripting Essentials
### DevOps / Cloud Engineer Quick Reference

This guide covers essential Linux commands used for **user management, system monitoring, networking troubleshooting, scripting, automation, and security**. It is designed as a practical reference for system administrators, DevOps engineers, and cloud professionals.

---

# 1. User & Group Management

Create, modify, and manage users and groups.

```bash
useradd <username>          # Create a new user account
userdel -r <username>       # Delete a user and their home directory
usermod -aG <group> <user>  # Add a user to a supplemental group
passwd <username>           # Set or change a user's password
groupadd <groupname>        # Create a new group
groupdel <groupname>        # Delete an existing group
```

View user information:

```bash
id                          # Display UID, GID, and group memberships
groups <username>           # Show groups a user belongs to
getent passwd <username>    # Retrieve user entry from system database
```

---

# 2. Identity & Session Information

Commands to identify current users and login activity.

```bash
whoami                      # Display current logged-in user
who                         # Show all logged-in users
w                           # Show logged-in users and their processes
last                        # Show login history
id                          # Show UID/GID of current user
```

---

# 3. Switching Users (su / sudo)

Commands for privilege escalation and switching identities.

```bash
su -                        # Switch to root with login shell
su <username>               # Switch to another user (same environment)
sudo -i                     # Open root login shell
sudo -u <user> <command>    # Run a command as another user
sudo !!                     # Run previous command with sudo
runuser -u <user> <cmd>     # Execute command as user (for scripts)
```

---

# 4. File Permissions & Ownership

Control file access and security.

```bash
chmod 755 file              # Change file permissions
chmod 600 file              # Restrict file access
chown user:group file       # Change file owner and group
chgrp group file            # Change file group ownership
ls -l                       # View file permissions
umask                       # Display default permission mask
```

Permission meaning:

| Permission | Numeric | Meaning |
|-----------|--------|--------|
| Read | 4 | View file contents |
| Write | 2 | Modify file |
| Execute | 1 | Run file |

Example:

```
chmod 755 script.sh
```

---

# 5. Bash Scripting Basics

Basic structure of a shell script.

```bash
#!/bin/bash
```

Example script:

```bash
#!/bin/bash

NAME=${1:-"Guest"}
CURRENT_DIR=$(pwd)

echo "Hello, $NAME"
echo "Current directory: $CURRENT_DIR"
```

Explanation:

| Syntax | Meaning |
|------|------|
| `#!/bin/bash` | Interpreter declaration |
| `$1` | First command-line argument |
| `$(command)` | Command substitution |
| `${var:-default}` | Default value if variable empty |

---

# 6. Bash Script Execution Rules

Before running a script:

```bash
chmod +x script.sh
```

Run script:

```bash
./script.sh
```

Important scripting rules:

- No spaces in variable assignments  
  ```
  VAR=value
  ```

- Use quotes around variables:

  ```
  "$VAR"
  ```

- Comments begin with:

  ```
  #
  ```

---

# 7. System Monitoring & Performance

Commands used to monitor system resources.

```bash
top                         # Real-time CPU and memory usage
htop                        # Interactive process viewer
free -h                     # Show RAM usage
df -h                       # Disk usage per filesystem
du -sh <path>               # Directory size summary
uptime                      # System uptime and load average
iostat                      # CPU and disk IO statistics
vmstat                      # Virtual memory statistics
```

Check system information:

```bash
uname -a                    # Kernel information
hostnamectl                 # System hostname and OS details
```

---

# 8. Process Management

Monitor and control running processes.

```bash
ps aux                      # List running processes
top                         # Monitor processes in real time
kill <PID>                  # Terminate process
kill -9 <PID>               # Force kill process
lsof -i :<port>             # Find process using a port
```

Example:

```bash
lsof -i :8080
```

---

# 9. Networking & Troubleshooting

Essential networking commands.

```bash
ip addr                     # Show network interfaces and IP addresses
ip route                    # Display routing table
ss -tulnp                   # Show open ports and listening services
netstat -tulnp              # List open network connections
ping <host>                 # Test network connectivity
traceroute <host>           # Trace packet route
```

Test remote connectivity:

```bash
curl -I https://example.com
```

Check port availability:

```bash
nc -zv <IP> <PORT>
```

DNS troubleshooting:

```bash
dig example.com
nslookup example.com
```

---

# 10. Log Monitoring & Analysis

Analyze logs for debugging and troubleshooting.

```bash
tail -f /var/log/syslog     # Monitor logs in real-time
grep "error" file.log       # Search log for pattern
journalctl -u nginx         # Show logs for a systemd service
journalctl -xe              # View detailed system logs
```

Text processing:

```bash
awk '{print $1}' file       # Extract column
sed 's/foo/bar/g' file      # Replace text
sort file | uniq -c         # Count duplicate lines
```

---

# 11. Service Management (systemd)

Manage services using systemctl.

```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
systemctl enable nginx
systemctl disable nginx
```

View running services:

```bash
systemctl list-units --type=service
```

---

# 12. Remote Access & File Transfer

Connect to remote servers and transfer files.

SSH login:

```bash
ssh user@server-ip
```

SSH using private key:

```bash
ssh -i key.pem user@server-ip
```

File transfer:

```bash
scp file user@host:/path
scp user@host:/file .
```

Efficient file synchronization:

```bash
rsync -avz source/ user@host:/destination
```

---

# 13. Task Scheduling (Cron)

Automate tasks using cron jobs.

Edit cron jobs:

```bash
crontab -e
```

View cron jobs:

```bash
crontab -l
```

Cron schedule format:

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week
│ │ │ └──── Month
│ │ └────── Day of month
│ └──────── Hour
└────────── Minute
```

Example:

```bash
0 2 * * * backup.sh
```

Runs every day at **2 AM**.

---

# 14. File Compression & Archives

Create and extract compressed files.

Create archive:

```bash
tar -czvf archive.tar.gz folder/
```

Extract archive:

```bash
tar -xzvf archive.tar.gz
```

Zip compression:

```bash
zip -r archive.zip folder/
```

---

# 15. DevOps Linux Workflow Example

Typical operations performed on a server:

```
SSH → Check system status → Monitor logs → Deploy application → Restart services
```

Example workflow:

```bash
ssh user@server
git pull
docker compose up -d
systemctl restart nginx
```

---

# 16. Quick DevOps Networking Checks

Useful for troubleshooting deployments.

Check open ports:

```bash
ss -tulnp
```

Test service response:

```bash
curl http://localhost:8080
```

Verify DNS:

```bash
dig domain.com
```

Check container ports:

```bash
docker ps
```

---

# 17. Recommended DevOps Linux Skills

Important areas every DevOps engineer should master:

- Linux user management
- Bash scripting
- Networking troubleshooting
- Process monitoring
- Log analysis
- Service management
- SSH and automation
- Cron scheduling
- File permissions and security
- Container debugging

---

# Summary

Linux administration forms the foundation of DevOps and cloud infrastructure. Key concepts include:

```
User Management
System Monitoring
Networking Troubleshooting
Service Management
Automation & Scripting
Security & Permissions
```

Mastering these commands significantly improves productivity when managing servers, containers, and cloud infrastructure.
