# 404 Page

Setting up a custom `404.html` page for static websites served by Nginx.

---

## **Overview**

1. **Create a custom `404.html` page** for your static website.
2. **Place the `404.html` page** in the appropriate directory.
3. **Modify the Nginx server block** to use the custom 404 page.
4. **Reload Nginx** to apply the changes.
5. **Test the configuration** to ensure it's working as expected.

---

## **Step 1: Create a Custom `404.html` Page**

First, we'll create a `404.html` page that will be displayed when a requested page is not found.

### **a. Navigate to Your Website's Root Directory**

For `example.com`, the root directory is `/var/www/example.com/html`.

```bash
cd /var/www/example.com/html
```

### **b. Create the `404.html` Page**

Use a text editor like `nano` to create the file.

```bash
nano 404.html
```

### **c. Add Content to `404.html`**

Add your custom HTML content for the 404 page. Here's a simple example:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>404 Not Found - example.com</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 100px; }
        h1 { font-size: 50px; color: #333; }
        p { font-size: 20px; color: #666; }
        a { color: #007BFF; text-decoration: none; }
        a:hover { text-decoration: underline; }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>Sorry, the page you're looking for doesn't exist.</p>
    <p><a href="/">Return to Home Page</a></p>
</body>
</html>
```

- **Customize the content** to match your website's style and branding.

### **d. Save and Exit**

Press `Ctrl + X`, then `Y`, and `Enter` to save and exit the editor.

---

## **Step 2: Modify the Nginx Server Block**

We need to configure Nginx to serve this custom `404.html` page when a 404 error occurs.

### **a. Open the Nginx Server Block Configuration**

Edit the server block file for `example.com`.

```bash
sudo nano /etc/nginx/sites-available/example.com
```

### **b. Add the `error_page` Directive**

Within the `server` block, add the following line (if it's not already present):

```nginx
error_page 404 /404.html;
```

This tells Nginx to use `/404.html` when a 404 error occurs.

### **c. Modify the `location /` Block (Optional)**

Ensure your `location /` block is correctly set up. It might already look like this:

```nginx
location / {
    try_files $uri $uri/ =404;
}
```

This configuration tries to serve the requested file; if it doesn't exist, Nginx returns a 404 error, which triggers the `error_page` directive.

### **d. Ensure the 404 Page is Served Correctly**

Add a `location` block for the `404.html` page to ensure it's served with the correct status code:

```nginx
location = /404.html {
    root /var/www/example.com/html;
    internal;
}
```

- **`root`**: Specifies the root directory for this location.
- **`internal`**: Prevents direct access to `/404.html` via the browser. It can only be served internally by Nginx.

### **e. Full Server Block Example**

Your server block should now look similar to this:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html index.htm;

    error_page 404 /404.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location = /404.html {
        internal;
    }

    # SSL configuration (if SSL is set up)
    # Include SSL directives here if applicable
}
```

### **f. Save and Exit**

Press `Ctrl + X`, then `Y`, and `Enter` to save and exit the editor.

---

## **Step 3: Test the Nginx Configuration**

Before reloading Nginx, test the configuration for syntax errors.

```bash
sudo nginx -t
```

If the configuration is correct, you should see:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

- **If there are errors**, Nginx will display messages indicating where the issue is. Go back and correct any mistakes.

---

## **Step 4: Reload Nginx**

Apply the changes by reloading Nginx.

```bash
sudo systemctl reload nginx
```

---

## **Step 5: Test the Custom 404 Page**

### **a. Clear Browser Cache (Optional)**

Sometimes browsers cache error pages. Clear your browser cache or use a private/incognito window.

### **b. Access a Non-Existent Page**

In your web browser, navigate to a URL on your domain that doesn't exist:

```
http://example.com/nonexistentpage
```

### **c. Verify the Custom 404 Page Displays**

- **Expected Result**: You should see your custom `404.html` page.
- **Check the Status Code**: Use browser developer tools or an online tool to verify that the HTTP status code is `404`.

---

## **Repeat the Process for Other Static Websites**

For any additional static websites hosted on your server, repeat these steps:

1. **Create a custom `404.html` page** in the site's root directory.
2. **Modify the Nginx server block** for that site.
3. **Reload Nginx**.
4. **Test the configuration**.

---

## **Additional Tips**

### **Customize Other Error Pages**

You can create custom pages for other HTTP errors (e.g., 403 Forbidden, 500 Internal Server Error):

- **Create the HTML files**: `403.html`, `500.html`, etc.
- **Add `error_page` directives**:

  ```nginx
  error_page 403 /403.html;
  error_page 500 502 503 504 /50x.html;
  ```

- **Add corresponding `location` blocks**:

  ```nginx
  location = /403.html {
      internal;
  }

  location = /50x.html {
      internal;
  }
  ```

### **Using a Shared Error Page Directory (Optional)**

If you want to use the same error pages across multiple sites:

1. **Create a Shared Directory**:

   ```bash
   sudo mkdir -p /var/www/error_pages
   ```

2. **Move Your `404.html` to the Shared Directory**:

   ```bash
   sudo mv /var/www/example.com/html/404.html /var/www/error_pages/
   ```

3. **Set Permissions**:

   ```bash
   sudo chmod -R 755 /var/www/error_pages
   ```

4. **Modify the `error_page` Directive**:

   In your server block, update the `error_page` path:

   ```nginx
   error_page 404 /error_pages/404.html;
   ```

5. **Adjust the `location` Block**:

   ```nginx
   location /error_pages/ {
       alias /var/www/error_pages/;
       internal;
   }
   ```

   - **`alias`**: Specifies the actual directory to serve files from.
   - **Ensure the `internal` directive** is present to prevent direct access.

6. **Repeat Steps in Other Server Blocks**:

   For other sites, you can reference the same shared error pages.

---

## **Ensuring Correct HTTP Status Codes**

- **Important**: The custom error pages should return the correct HTTP status codes (e.g., 404 for not found).
- **Using the `internal` Directive**: Ensures that Nginx serves the error page internally while preserving the original status code.

---

## **Troubleshooting**

- **Custom 404 Page Not Displaying**:

  - Check file paths in the `error_page` and `location` directives.
  - Ensure the `404.html` file exists in the specified location.

- **Status Code is 200 Instead of 404**:

  - Ensure you're using `=404` in the `try_files` directive:

    ```nginx
    try_files $uri $uri/ =404;
    ```

  - Avoid redirecting to the `404.html` file directly in the `try_files` directive, as this may change the status code to `200`.

- **Nginx Configuration Test Fails**:

  - Review the error messages for line numbers and syntax issues.
  - Ensure all brackets `{}` are properly closed.

---

## **Summary**

- **Created a custom `404.html` page** in your website's root directory.
- **Modified the Nginx server block**:

  - Added the `error_page` directive.
  - Configured a `location` block for the error page.

- **Reloaded Nginx** to apply the changes.
- **Tested the configuration** by accessing a non-existent page and verifying that the custom 404 page is displayed with the correct HTTP status code.

---

## **Next Steps**

- **Monitor Logs**: Keep an eye on your Nginx access and error logs to identify broken links or frequent 404 errors.

  ```bash
  sudo tail -f /var/log/nginx/access.log
  sudo tail -f /var/log/nginx/error.log
  ```

- **Implement Redirects**: For frequently requested pages that no longer exist, consider setting up redirects to relevant content.

- **Enhance the 404 Page**:

  - Add a search bar to help users find what they're looking for.
  - Include links to popular pages or the sitemap.

- **Maintain Consistency**: Ensure that the style of your error pages matches the rest of your website for a seamless user experience.
