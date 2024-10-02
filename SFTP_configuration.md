# SFTP configuration

Step-by-step guide on how to set up SFTP access to server, allowing the user `ftpuser00` to access the `/var/www/example.com/html` directory securely. This setup ensures that the user can only access the specified directory and cannot navigate to other parts of the server.

## **Overview**

1. **Create an SFTP group** for users who need SFTP access.
2. **Create the user `ftpuser00`**, assign them to the SFTP group, and set their home directory.
3. **Configure directory permissions** to secure the SFTP environment.
4. **Modify the SSH configuration** to restrict SFTP users to their home directories.
5. **Restart the SSH service** to apply changes.
6. **Test the SFTP connection**.

---

## **Step 1: Create an SFTP Group**

Create a new group called `sftpusers` to manage all SFTP users.

```bash
sudo groupadd sftpusers
```

---

## **Step 2: Create the User `ftpuser00`**

Create a new user, assign them to the `sftpusers` group, and set their home directory to `/var/www/example.com`.

```bash
sudo useradd -m -d /var/www/example.com -G sftpusers -s /usr/sbin/nologin ftpuser00
```

- **`-m`**: Creates the home directory if it doesn't exist.
- **`-d`**: Specifies the user's home directory.
- **`-G`**: Assigns the user to the specified group.
- **`-s /usr/sbin/nologin`**: Disables shell access for security.

### **Set the User's Password**

```bash
sudo passwd ftpuser00
```

Enter a strong password when prompted.

---

## **Step 3: Configure Directory Permissions**

### **a. Set Ownership of the Chroot Directory**

The chroot directory (`/var/www/example.com`) must be owned by `root` and not writable by any other user or group.

```bash
sudo chown root:root /var/www/example.com
sudo chmod 755 /var/www/example.com
```

### **b. Set Ownership of the `html` Directory**

Grant ownership of the `html` directory to `ftpuser00` so they can read and write files.

```bash
sudo chown ftpuser00:sftpusers /var/www/example.com/html
sudo chmod 755 /var/www/example.com/html
```

### **Directory Structure and Permissions Recap**

- **`/var/www/example.com`**: Owned by `root:root` with `755` permissions.
- **`/var/www/example.com/html`**: Owned by `ftpuser00:sftpusers` with `755` permissions.

---

## **Step 4: Modify SSH Configuration**

Edit the SSH daemon configuration file to set up the SFTP environment.

```bash
sudo nano /etc/ssh/sshd_config
```

### **a. Disable Subsystem sftp (Optional)**

Ensure the following line is present (it usually is by default):

```ssh
Subsystem sftp internal-sftp
```

### **b. Add Configuration for the SFTP Group**

At the end of the file, add:

```ssh
Match Group sftpusers
    ChrootDirectory %h
    ForceCommand internal-sftp
    AllowTCPForwarding no
    X11Forwarding no
```

- **`Match Group sftpusers`**: Applies the following rules to users in the `sftpusers` group.
- **`ChrootDirectory %h`**: Restricts users to their home directory.
- **`ForceCommand internal-sftp`**: Forces the use of SFTP and disables shell access.
- **`AllowTCPForwarding no`** and **`X11Forwarding no`**: Enhances security by disabling unnecessary features.

### **c. Save and Exit**

- Press `Ctrl + X`, then `Y`, and `Enter` to save the changes.

---

## **Step 5: Restart the SSH Service**

Apply the changes by restarting the SSH daemon.

```bash
sudo systemctl restart sshd
```

---

## **Step 6: Test the SFTP Connection**

### **a. From a Local Machine**

Use an SFTP client like **FileZilla**, **WinSCP**, or the command line `sftp` tool.

#### **Using FileZilla**

1. **Open FileZilla**.
2. **Create a New Site**:
   - **Host**: `example.com`
   - **Port**: `22`
   - **Protocol**: `SFTP - SSH File Transfer Protocol`
   - **Logon Type**: `Normal`
   - **User**: `ftpuser00`
   - **Password**: *The password you set earlier*
3. **Connect**.

#### **Using Command Line**

```bash
sftp ftpuser00@example.com
```

Enter the password when prompted.

### **b. Verify Access**

- Upon successful connection, you should be in the root of the chroot directory, which appears as the root (`/`).
- List the files and directories:

  ```bash
  ls
  ```

  You should see the `html` directory.

- Navigate to the `html` directory:

  ```bash
  cd html
  ```

- You can now upload, download, and manage files within this directory.

---

## **Step 7: Optional Security Enhancements**

### **a. Restrict SSH Access Further**

If you want to ensure that only specific users can SSH into the server:

- **Edit the SSH Configuration**:

  ```bash
  sudo nano /etc/ssh/sshd_config
  ```

- **Add or Modify the `AllowUsers` Directive**:

  ```ssh
  AllowUsers your_admin_user
  ```

  Replace `your_admin_user` with your administrative username.

- **Save and Restart SSH**:

  ```bash
  sudo systemctl restart sshd
  ```

### **b. Use SSH Keys Instead of Passwords**

For enhanced security, you can set up SSH key authentication for administrative users and disable password authentication.

---

## **Summary**

- **User Creation**: Created `ftpuser00` with no shell access, assigned to `sftpusers`.
- **Permissions**: Set strict permissions on `/var/www/example.com` and appropriate ownership on `/html`.
- **SSH Configuration**: Modified `sshd_config` to chroot SFTP users to their home directories.
- **Testing**: Confirmed SFTP access is restricted to the intended directory.

---

## **Adding More SFTP Users**

To add more users with similar access:

1. **Create the User**:

   ```bash
   sudo useradd -m -d /path/to/chroot -G sftpusers -s /usr/sbin/nologin username
   ```

2. **Set the User's Password**:

   ```bash
   sudo passwd username
   ```

3. **Configure Directory Permissions**:

   - Set the chroot directory to be owned by `root:root` with `755` permissions.
   - Assign ownership of subdirectories to the new user.

4. **Ensure SSH Configuration Matches**:

   - No changes needed if they are part of the `sftpusers` group.

---

## **Notes and Best Practices**

- **Chroot Directory Ownership**: The chroot directory must be owned by `root` and not writable by any other user for security reasons.
- **Subdirectories**: Users can have write permissions within subdirectories they own.
- **Security**: Always use SFTP over FTP, as FTP transmits data in plaintext, including passwords.
- **Firewall Settings**: Ensure that port `22` (SSH) is open if you have a firewall configured.

  ```bash
  sudo ufw allow OpenSSH
  ```

- **Monitoring**: Regularly check `/var/log/auth.log` for any suspicious login attempts.

---

## **Troubleshooting**

- **Permission Denied Errors**: Check directory ownership and permissions. The chroot directory must be owned by `root:root` with `755` permissions.
- **Cannot Access Subdirectories**: Ensure the user owns the subdirectories and has the necessary permissions.
- **SSH Service Fails to Restart**: If there are syntax errors in `sshd_config`, the SSH service will fail to restart. Use `sudo sshd -t` to test the configuration before restarting.

---

## **Next Steps**

- **Regular Updates**: Keep your server updated.

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- **Backups**: Regularly back up your website files and configurations.
- **Security Audits**: Periodically review user access and SSH configurations.
- **Documentation**: Keep a record of users and permissions for future reference.

---