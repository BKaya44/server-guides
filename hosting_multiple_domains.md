# Hosting multiple domains

Step by step on how to configure VPS to host multiple domains, including static websites and Node.js applications.

---

## **Overview**

1. **Ensure server is up to date** making sure you have the latest version of your server.
2. **Install Nginx** Nginx is a commonly used HTTP web server.
3. **Adjusting Firewall** to secure the server environment.
4. **Configure DNS records** to redirect your domain to the correct server.
5. **Setting up directories** to showcase your website on the correct domain.
6. **Testing domain**.

---
## **Step 1: Update and Upgrade Your Server**

First, ensure your server is up to date.

```bash
sudo apt update
sudo apt upgrade -y
```

---

## **Step 2: Install Nginx**

Install Nginx, which will serve your websites.

```bash
sudo apt install nginx -y
```

---

## **Step 3: Adjust the Firewall**

If you have UFW (Uncomplicated Firewall) enabled, allow Nginx traffic.

```bash
sudo ufw allow 'Nginx Full'
```

---

## **Step 4: Configure DNS Records**

1. **Log in to Your Domain Registrar**

   Access the DNS management panel where you purchased your domain.

2. **Create an 'A' Record**

   - **Host/Name**: `@` (or leave it blank)
   - **Type**: `A`
   - **Value/Points to**: *Your VPS Public IP Address*
   - **TTL**: Default or 3600 seconds

3. **Add a 'www' CNAME Record (Optional)**

   - **Host/Name**: `www`
   - **Type**: `CNAME`
   - **Value/Points to**: `example.com`
   - **TTL**: Default

**Note**: DNS changes can take up to 24 hours to propagate, but usually, it's faster.

---

## **Step 5: Set Up Directory Structure for Your Domain**

Create a directory to hold your website files.

```bash
sudo mkdir -p /var/www/example.com/html
```

Set the ownership to your current user for easier file management.

```bash
sudo chown -R $USER:$USER /var/www/example.com/html
```

Set the correct permissions.

```bash
sudo chmod -R 755 /var/www/example.com
```

---

## **Step 6: Upload Your Static Website Files**

Place your HTML, CSS, and JS files into the directory.

```bash
cp -r /path/to/your/website/* /var/www/example.com/html/
```

---

## **Step 7: Create an Nginx Server Block for example.com**

Create a new server block configuration file.

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Add the following configuration:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html index.htm index.nginx-debian.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Save and exit the editor (`Ctrl+X`, then `Y`, then `Enter`).

---

## **Step 8: Enable the Server Block**

Create a symbolic link to enable the server block.

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

---

## **Step 9: Test and Reload Nginx**

Test the configuration for syntax errors.

```bash
sudo nginx -t
```

If the test is successful, reload Nginx.

```bash
sudo systemctl reload nginx
```

---

## **Step 10: Secure Your Website with SSL (Optional but Recommended)**

Install Certbot and the Nginx plugin.

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Obtain and install the SSL certificate.

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Follow the prompts to agree to the terms and input your email address.

---

## **Step 11: Verify Your Website**

Visit `http://example.com` or `https://example.com` in your web browser to see your static website.

---

## **Step 12: Adding More Domains and Static Websites**

To add more domains and websites, repeat the following steps for each new domain.

### **a. Configure DNS Records for the New Domain**

Set up 'A' records pointing to your server's IP.

### **b. Create Directory Structure**

```bash
sudo mkdir -p /var/www/yournewdomain.com/html
sudo chown -R $USER:$USER /var/www/yournewdomain.com/html
sudo chmod -R 755 /var/www/yournewdomain.com
```

### **c. Upload Your Website Files**

```bash
cp -r /path/to/your/newwebsite/* /var/www/yournewdomain.com/html/
```

### **d. Create a New Nginx Server Block**

```bash
sudo nano /etc/nginx/sites-available/yournewdomain.com
```

Add the configuration:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name yournewdomain.com www.yournewdomain.com;

    root /var/www/yournewdomain.com/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### **e. Enable the Server Block**

```bash
sudo ln -s /etc/nginx/sites-available/yournewdomain.com /etc/nginx/sites-enabled/
```

### **f. Test and Reload Nginx**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### **g. Secure the New Domain with SSL**

```bash
sudo certbot --nginx -d yournewdomain.com -d www.yournewdomain.com
```

---

## **Step 13: Preparing for Node.js Applications**

### **a. Install Node.js and npm**

Install Node.js from the NodeSource repository for the latest version.

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
```

Verify the installation.

```bash
node -v
npm -v
```

### **b. Set Up a Sample Node.js Application**

Create a directory for your Node.js app.

```bash
mkdir ~/my-node-app
cd ~/my-node-app
```

Initialize a new Node.js project.

```bash
npm init -y
```

Install Express.js (optional, but common for web apps).

```bash
npm install express
```

Create an `app.js` file.

```bash
nano app.js
```

Add a simple application code:

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => res.send('Hello from Node.js App!'));

app.listen(port, () => console.log(`App listening at http://localhost:${port}`));
```

### **c. Test the Node.js Application**

Run the app.

```bash
node app.js
```

Visit `http://your_server_ip:3000` to see the message.

### **d. Set Up Nginx as a Reverse Proxy**

Create a server block for your Node.js app.

```bash
sudo nano /etc/nginx/sites-available/nodeappdomain.com
```

Add the following configuration:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name nodeappdomain.com www.nodeappdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
}
```

Enable the server block.

```bash
sudo ln -s /etc/nginx/sites-available/nodeappdomain.com /etc/nginx/sites-enabled/
```

Test and reload Nginx.

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### **e. Secure the Node.js Domain with SSL**

```bash
sudo certbot --nginx -d nodeappdomain.com -d www.nodeappdomain.com
```

### **f. Use a Process Manager for Node.js (Recommended)**

Install PM2 to manage your Node.js application.

```bash
sudo npm install -g pm2
```

Start your app with PM2.

```bash
pm2 start app.js
```

Set PM2 to start on boot.

```bash
pm2 startup systemd
pm2 save
```

---

## **Step 14: Managing Multiple Websites and Applications**

- **Static Websites**: For each static website, create a new directory, server block, and set up SSL.
- **Node.js Applications**: For each Node.js app, run it on a different port and configure Nginx to reverse proxy to it.

---

## **Step 15: Optional Enhancements**

### **a. Use Domains Without 'www'**

If you prefer to force `www` or non-`www`, you can set up redirects in your server blocks.

### **b. Automate with Scripts or Tools**

Consider using deployment scripts or tools like **Ansible** for automation.

### **c. Monitor Your Server**

Install tools like **htop**, **netdata**, or **Nagios** for monitoring.

---

## **Future Additions**

- When adding new domains or applications, remember to:

  - Configure DNS records accordingly.
  - Set up directories and server blocks.
  - Secure with SSL certificates.
  - Use different ports for multiple Node.js apps and configure Nginx accordingly.

- Keep your system updated regularly:

  ```bash
  sudo apt update
  sudo apt upgrade -y
  ```

- Backup your configurations and website data periodically.

---
