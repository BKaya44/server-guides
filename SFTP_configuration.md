# Complete SFTP Server Configuration Guide

A comprehensive guide to securely configure SFTP access on your server, enabling restricted file access for web developers and content managers while maintaining robust security.

---

## Initial Setup

### Step 1: Create SFTP User Group

Create a dedicated group for all SFTP users:

```bash
sudo groupadd sftponly
```

### Step 2: Verify OpenSSH Version

Ensure you have a compatible OpenSSH version (5.2+):

```bash
ssh -V
```

If your version is older than 5.2, update OpenSSH before proceeding.

---

## User Configuration

### Step 3: Create SFTP User

Create the user `ftpuser00` with secure defaults:

```bash
# Create user with no shell access and custom home directory
sudo useradd \
  --create-home \
  --home-dir /var/www/example.com \
  --groups sftponly \
  --shell /usr/sbin/nologin \
  --comment "SFTP User for example.com" \
  ftpuser00
```

**Parameter Explanation:**
- `--create-home`: Creates the home directory if it doesn't exist
- `--home-dir`: Sets custom home directory path
- `--groups`: Adds user to the SFTP group
- `--shell`: Disables shell access for security
- `--comment`: Adds descriptive comment

### Step 4: Set Strong Password

```bash
sudo passwd ftpuser00
```

**Password Requirements:**
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, and symbols
- No dictionary words or personal information

### Step 5: Configure Directory Structure and Permissions

The chroot environment requires specific ownership and permissions:

#### Set Chroot Directory (Critical Security Step)

```bash
# The chroot directory MUST be owned by root and not writable by others
sudo chown root:root /var/www/example.com
sudo chmod 755 /var/www/example.com
```

#### Configure Target Directory

```bash
# Create html directory if it doesn't exist
sudo mkdir -p /var/www/example.com/html

# Set appropriate ownership and permissions
sudo chown ftpuser00:sftponly /var/www/example.com/html
sudo chmod 755 /var/www/example.com/html
```

#### Create Additional Directories (Optional)

```bash
# Create common web directories
sudo mkdir -p /var/www/example.com/html/{css,js,images,uploads}
sudo chown -R ftpuser00:sftponly /var/www/example.com/html/
sudo chmod -R 755 /var/www/example.com/html/
```

**Directory Structure:**
```
/var/www/example.com/          (root:root, 755) - Chroot directory
├── html/                      (ftpuser00:sftponly, 755) - Writable area
│   ├── css/
│   ├── js/
│   ├── images/
│   └── uploads/
```

---

## SSH Daemon Configuration

### Step 6: Backup Current SSH Configuration

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d)
```

### Step 7: Configure SSH for SFTP

Edit the SSH daemon configuration:

```bash
sudo nano /etc/ssh/sshd_config
```

#### Verify Internal SFTP Subsystem

Ensure this line exists (usually present by default):

```ssh
Subsystem sftp internal-sftp
```

#### Add SFTP Group Configuration

Add the following at the end of the file:

```ssh
# SFTP-only users configuration
Match Group sftponly
    # Restrict to home directory
    ChrootDirectory %h
    
    # Force SFTP-only access
    ForceCommand internal-sftp
    
    # Disable potentially dangerous features
    AllowTCPForwarding no
    X11Forwarding no
    AllowAgentForwarding no
    
    # Optional: Limit connections
    MaxSessions 5
    MaxStartups 3
    
    # Enhanced logging
    LogLevel VERBOSE
```

**Configuration Explained:**
- `ChrootDirectory %h`: Jails user to their home directory
- `ForceCommand internal-sftp`: Only allows SFTP, no shell commands
- `AllowTCPForwarding no`: Prevents port forwarding
- `X11Forwarding no`: Disables X11 GUI forwarding
- `AllowAgentForwarding no`: Prevents SSH agent forwarding
- `MaxSessions/MaxStartups`: Limits concurrent connections

### Step 8: Validate SSH Configuration

Test the configuration before applying:

```bash
sudo sshd -t
```

If no errors are reported, the configuration is valid.

### Step 9: Apply Configuration

Restart the SSH service:

```bash
# For systemd systems (Ubuntu 16.04+, CentOS 7+)
sudo systemctl restart sshd

# For older systems
sudo service ssh restart
```

Verify the service is running:

```bash
sudo systemctl status sshd
```

---

## Testing and Validation

### Step 10: Test SFTP Connection

#### Command Line Test

```bash
# Test from another machine or local terminal
sftp ftpuser00@your-server-ip

# After connecting, test basic commands:
pwd          # Should show / (chroot root)
ls           # Should show html directory
cd html      # Navigate to writable area
put test.txt # Upload a test file
ls           # Verify file was uploaded
quit         # Exit SFTP
```

#### FileZilla Configuration

1. **Host**: `your-server-ip` or `example.com`
2. **Protocol**: `SFTP - SSH File Transfer Protocol`
3. **Port**: `22`
4. **Logon Type**: `Normal`
5. **User**: `ftpuser00`
6. **Password**: Your set password

#### WinSCP Configuration (Windows)

1. **File Protocol**: `SFTP`
2. **Host Name**: `your-server-ip`
3. **Port**: `22`
4. **User Name**: `ftpuser00`
5. **Password**: Your set password

### Step 11: Verify Security Restrictions

Test that security measures are working:

```bash
# This should fail (no shell access)
ssh ftpuser00@your-server-ip

# SFTP should work but be restricted
sftp ftpuser00@your-server-ip
cd /etc    # Should fail - cannot escape chroot
cd ../     # Should fail - cannot go above chroot
```

---

## Advanced Security

### Step 12: Implement SSH Key Authentication (Recommended)

For enhanced security, use SSH keys instead of passwords:

#### Generate SSH Key Pair (on client machine)

```bash
ssh-keygen -t ed25519 -C "ftpuser00@example.com" -f ~/.ssh/ftpuser00_key
```

#### Install Public Key on Server

```bash
# Create .ssh directory in user's home
sudo mkdir -p /var/www/example.com/.ssh
sudo chmod 700 /var/www/example.com/.ssh

# Add public key
sudo nano /var/www/example.com/.ssh/authorized_keys
# Paste the public key content here

# Set proper ownership and permissions
sudo chown -R ftpuser00:sftponly /var/www/example.com/.ssh
sudo chmod 600 /var/www/example.com/.ssh/authorized_keys
```

#### Disable Password Authentication (Optional)

In `/etc/ssh/sshd_config`, add to the Match Group section:

```ssh
Match Group sftponly
    # ... existing configuration ...
    PasswordAuthentication no
    PubkeyAuthentication yes
```

### Step 13: Configure Firewall

Ensure SSH port is properly configured:

```bash
# For UFW (Ubuntu)
sudo ufw allow OpenSSH
sudo ufw enable

# For firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# For iptables
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

### Step 14: Set Up Log Monitoring

Monitor SFTP access in system logs:

```bash
# Monitor real-time SFTP connections
sudo tail -f /var/log/auth.log | grep sftp

# Check for failed login attempts
sudo grep "Failed password" /var/log/auth.log | tail -10
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue: "Permission denied" on connection

**Possible Causes:**
- Incorrect chroot directory ownership
- Wrong permissions on chroot directory
- User not in correct group

**Solution:**
```bash
# Verify and fix chroot ownership
sudo chown root:root /var/www/example.com
sudo chmod 755 /var/www/example.com

# Verify user group membership
groups ftpuser00
```

#### Issue: "This service allows sftp connections only"

**Cause:** User trying to SSH instead of SFTP (this is expected behavior)

**Solution:** Use SFTP client instead of SSH

#### Issue: SSH service fails to restart

**Cause:** Syntax error in sshd_config

**Solution:**
```bash
# Test configuration
sudo sshd -t

# Restore backup if needed
sudo cp /etc/ssh/sshd_config.backup.* /etc/ssh/sshd_config
```

#### Issue: Cannot write files in SFTP

**Possible Causes:**
- Incorrect ownership of target directory
- Insufficient permissions
- Disk space full

**Solution:**
```bash
# Check ownership and permissions
ls -la /var/www/example.com/

# Fix ownership if needed
sudo chown ftpuser00:sftponly /var/www/example.com/html

# Check disk space
df -h
```

### Debug Mode

Enable debug mode for detailed troubleshooting:

```bash
# Run SSH daemon in debug mode (temporary)
sudo /usr/sbin/sshd -d -p 2222

# Connect using debug port
sftp -P 2222 ftpuser00@localhost
```

---

## Maintenance

### Adding Additional Users

To add more SFTP users with similar access:

```bash
# Create new user
sudo useradd -m -d /var/www/newsite.com -G sftponly -s /usr/sbin/nologin ftpuser01

# Set password
sudo passwd ftpuser01

# Configure directories
sudo chown root:root /var/www/newsite.com
sudo chmod 755 /var/www/newsite.com
sudo mkdir -p /var/www/newsite.com/html
sudo chown ftpuser01:sftponly /var/www/newsite.com/html
sudo chmod 755 /var/www/newsite.com/html
```

### User Management Commands

```bash
# List all SFTP users
getent group sftponly

# Remove SFTP user
sudo userdel -r ftpuser00

# Disable user temporarily
sudo usermod -L ftpuser00

# Enable user
sudo usermod -U ftpuser00

# Change user's directory
sudo usermod -d /var/www/newpath ftpuser00
```

### Regular Security Checks

#### Monthly Tasks:
- Review `/var/log/auth.log` for suspicious activity
- Update system packages: `sudo apt update && sudo apt upgrade`
- Check for unused SFTP accounts
- Verify directory permissions haven't changed

#### Quarterly Tasks:
- Rotate SSH host keys if required
- Review and update user access requirements
- Test backup and restore procedures
- Update passwords or SSH keys

### Monitoring Script

Create a simple monitoring script:

```bash
#!/bin/bash
# /usr/local/bin/sftp-monitor.sh

echo "SFTP Users Status Report - $(date)"
echo "=================================="

echo "Active SFTP users:"
getent group sftponly | cut -d: -f4 | tr ',' '\n'

echo -e "\nRecent SFTP connections:"
grep "sftp-server" /var/log/auth.log | tail -5

echo -e "\nDisk usage for SFTP directories:"
du -sh /var/www/*/html 2>/dev/null

echo -e "\nSSH service status:"
systemctl is-active sshd
```

Make it executable and run monthly:

```bash
sudo chmod +x /usr/local/bin/sftp-monitor.sh
sudo crontab -e
# Add: 0 9 1 * * /usr/local/bin/sftp-monitor.sh | mail -s "SFTP Status" admin@example.com
```

---
