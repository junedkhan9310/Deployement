## Deploying a MERN Stack Project on Hostinger VPS

### Table of Contents
1. [Preparing the VPS Environment](#1-preparing-the-vps-environment)
2. [Deploying the React Frontend](#2-deploying-the-react-frontend)
3. [Configuring Nginx as a Reverse Proxy](#3-configuring-nginx-as-a-reverse-proxy)
4. [Setting Up SSL Certificates](#4-setting-up-ssl-certificates)
5. [Additional Recommendations](#5-additional-recommendations)
6. [Troubleshooting Tips](#6-troubleshooting-tips)
7. [Contact for Further Assistance](#7-contact-for-further-assistance)

---

### 1. Preparing the VPS Environment

#### **1.1. Get Your VPS Hosting**
You can purchase your VPS hosting from [Hostinger VPS](https://greatstack.dev/go/hostinger-vps).

#### **1.2. Access Your VPS via Terminal**
Use SSH to log in to your VPS. Replace `user` and `your_vps_ip` with your actual username and VPS IP address.

```bash
ssh user@your_vps_ip

```

#### **1.3. Install Essential Tools (If Not Present)**
Ensure that Git, Node.js, npm, and PM2 are installed. If they are already installed, you can skip these steps.

```bash
# Install Git
sudo apt install git -y

# Install Node.js and npm (using NodeSource)
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node -v
npm -v

# Install PM2 globally
sudo npm install -g pm2
```

---

### 2. Deploying the React Frontend

#### **2.1. Clone Your Frontend Repository**
Assuming your frontend is part of a repository. Replace the URL with your actual repository link.

```bash
git clone https://github.com/yourusername/your-frontend-repo.git
cd your-frontend-repo/frontend
```

#### **2.2. Install Dependencies**
Install the necessary npm packages. Ensure you run this command in the `frontend` directory.

```bash
npm install
```

*Note:* The original guide included commands like `npm install force` and `npm install peer legacy deps`, which are not standard npm commands. If specific packages are required, install them individually:

```bash
npm install <package-name>
```

#### **2.3. Create a Build of the Project**
Generate the production build of your React application.

```bash
npm run build
```

#### **2.4. Serve the Frontend with PM2**
Use a static server like `serve` to host your React build. If `serve` is not installed, install it globally.

```bash
# Install 'serve' globally if not already installed
sudo npm install -g serve

# Start the frontend
pm2 serve build 3000 --name "frontend" --single
```

*Alternatively, if you have a custom server setup:*

```bash
pm2 start npm --name "frontend" -- start
```

#### **2.5. Save PM2 Process List and Enable Startup Script**
Ensure that PM2 restarts your application on server reboot.

```bash
pm2 save
pm2 startup
```

*Follow the instructions provided by the `pm2 startup` command to enable PM2 on system boot.*

---

### 3. Configuring Nginx as a Reverse Proxy

#### **3.1. Install Nginx (If Not Present)**
If Nginx is not already installed on your VPS, install it.

```bash
sudo apt install nginx -y
```

#### **3.2. Configure DNS Records**
Ensure your domain and any subdomains point to your VPS IP by setting up A records in your DNS settings.

- **Primary Domain (e.g., yourdomain.com):**
  - **Type:** A
  - **Name:** @
  - **Value:** `your_vps_ip`
  - **TTL:** Automatic or as desired

- **Subdomain (e.g., education.yourdomain.com):**
  - **Type:** A
  - **Name:** education
  - **Value:** `your_vps_ip`
  - **TTL:** Automatic or as desired

#### **3.3. Create Nginx Configuration Files**
Navigate to Nginx's configuration directory and create a configuration file for your frontend.

```bash
cd /etc/nginx/conf.d
```

##### **3.3.1. Configure Frontend**

Create a new Nginx configuration file for your frontend. Replace `yourfrontend.com` with your actual domain and adjust paths as necessary.

```bash
sudo nano yourfrontend.com.conf
```

**Example Configuration:**

```nginx
# HTTPS server block (Port 443)
server {
    listen 443 ssl;
    server_name yourfrontend.com www.yourfrontend.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/yourfrontend.com/fullchain.pem;  # Replace with your SSL paths
    ssl_certificate_key /etc/letsencrypt/live/yourfrontend.com/privkey.pem;  # Replace with your SSL paths
    include /etc/letsencrypt/options-ssl-nginx.conf;  # Use Certbot's SSL settings
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;  # Use Certbot's DH parameters

    # Serve the React build files
    root /home/deployer/your-frontend-repo/frontend/build;
    index index.html index.htm;

    location / {
        try_files $uri /index.html;
    }

    access_log /var/log/nginx/yourfrontend_access.log;
    error_log /var/log/nginx/yourfrontend_error.log;
}

# HTTP server block (Port 80) for redirection to HTTPS
server {
    listen 80;
    server_name yourfrontend.com www.yourfrontend.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}
```

*Ensure that the `root` directive points to the correct path where your React build resides.*

#### **3.4. Test and Reload Nginx**

Test the Nginx configuration for syntax errors.

```bash
sudo nginx -t
```

If the test is successful, reload Nginx to apply the changes.

```bash
sudo systemctl reload nginx
```

---

### 4. Setting Up SSL Certificates

#### **4.1. Install Certbot (If Not Present)**
If Certbot isn't already installed on your VPS, install it to obtain SSL certificates.

```bash
sudo apt install -y certbot python3-certbot-nginx
```

#### **4.2. Obtain SSL Certificates**
Use Certbot to obtain and install SSL certificates for your domain.

```bash
sudo certbot --nginx -d yourfrontend.com -d www.yourfrontend.com
```

*Follow the interactive prompts to complete the SSL setup.*

#### **4.3. Verify Auto-Renewal**
Certbot sets up a cron job for automatic renewal. To test it:

```bash
sudo certbot renew --dry-run
```

*Ensure no errors are reported.*

---

### 5. Additional Recommendations

1. **Firewall Configuration:**
   Ensure that your VPS firewall allows necessary traffic.

   ```bash
   sudo ufw allow OpenSSH
   sudo ufw allow 'Nginx Full'
   sudo ufw enable
   ```

2. **Environment Variables Management:**
   Use environment variables for any sensitive information required by your frontend. This can be managed through build-time variables or server-side configurations.

3. **Monitoring and Logging:**
   - Utilize PM2's monitoring tools.
   - Set up log rotation for Nginx logs to prevent disk space issues.

4. **Continuous Deployment:**
   For smoother deployments, consider integrating CI/CD pipelines using tools like GitHub Actions, Jenkins, or GitLab CI.

5. **Backup Strategy:**
   Regularly back up your code, configurations, and any other essential data.

---

### 6. Troubleshooting Tips

- **Nginx Errors:**
  - Check Nginx logs located in `/var/log/nginx/yourfrontend_error.log`.
  - Ensure that the server blocks have correct syntax and paths.

- **PM2 Issues:**
  - Use `pm2 status` to check running processes.
  - Restart the frontend process with `pm2 restart frontend` if necessary.

- **SSL Problems:**
  - Ensure DNS records are correctly pointing to your VPS.
  - Verify that SSL certificates are correctly installed and not expired.

- **React Build Issues:**
  - Ensure that the `npm run build` command completes successfully.
  - Verify that the `root` path in the Nginx configuration points to the correct build directory.

---

### 7. Contact for Further Assistance

If you encounter issues or need personalized support during deployment, feel free to reach out:

- **Email:** [junedkhanfarh@gmail.com](mailto:junedkhanfarh@gmail.com)


---

