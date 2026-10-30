# VPS Hardening & Web Application Security Guide

A comprehensive guide for developers deploying web applications on Linux VPS instances running Nginx, Node.js, and frontend frameworks like React or Vite.

---

## 1. Introduction & Core Security Model

When you deploy a application to a public VPS, automated internet bots immediately begin scanning your IP address for common configuration files and vulnerabilities. Protecting your server requires a multi-layered defense strategy:

1. **Physical Isolation:** Keeping sensitive configuration files out of public web directories.
2. **Web Server Rules:** Hardening Nginx to explicitly drop requests for hidden files and sensitive file extensions.
3. **Environment Security:** Understanding how frontend build tools handle environment variables vs. backend services.

---

## 2. File System Architecture & Permissions

### Never Store Secrets in Web Root
Your public web directory (`/var/www/html`) should **only** contain public static files (HTML, CSS, compiled JavaScript, images).

```text
/home/ubuntu/
└── apps/
    └── my-app/                <-- Private Application Directory
        ├── .env               <-- Stored safely outside web root
        ├── docker-compose.yml
        ├── serverless.yaml
        ├── package.json
        └── src/
/var/
└── www/
    └── html/                  <-- Public Web Root
        ├── index.html         <-- Public compiled assets only
        └── assets/
```

### File Permission Hardening
Restrict file permissions on configuration files so only the system owner can read them:

```bash
# Restrict read/write access to owner only
chmod 600 /home/ubuntu/apps/my-app/.env
chmod 600 /home/ubuntu/apps/my-app/config.json

# Restrict directory access
chmod 700 /home/ubuntu/apps/my-app
```

---

## 3. Nginx Server Hardening

Configure Nginx to globally deny requests attempting to access hidden files, version control assets, or configuration files.

Add the following directives inside your server block (typically in `/etc/nginx/sites-available/your-site`):

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    root /var/www/html;

    # 1. Deny access to hidden files/directories (e.g., .git, .env, .npmrc)
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 2. Block access to common backend config and script files
    location ~* \.(env|json|yaml|yml|toml|ini|lock|log|sh|conf)$ {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 3. Protect specific sensitive endpoints explicitly
    location ~* /(docker-compose|serverless|samconfig|appsettings) {
        deny all;
        access_log off;
        log_not_found off;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Validate and Reload Nginx
Always test syntax before reloading Nginx configurations:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. Securing React & Frontend Applications

### Development Servers vs. Production Builds
Never run development servers (`npm start`, `vite`, `next dev`) on a live production environment.

| Feature | Development Server (`npm start`) | Production Build (`npm run build`) |
| :--- | :--- | :--- |
| **Purpose** | Local debugging and hot reloading | Optimized static assets for distribution |
| **Performance** | High RAM usage, single-threaded | Extremely fast, handled efficiently by Nginx |
| **Security Risk** | Exposes unminified code and dev ports | Minified code, no active server processes |

### Frontend `.env` Exposure Risk
Environment variables embedded in frontend tools (e.g., `REACT_APP_` or `VITE_`) are **injected directly into the bundled client-side JavaScript file**. 

* **Rule:** Anyone inspecting the browser page source can view client-side variables.
* **Best Practice:** Never put private API keys, database credentials, or backend tokens in frontend `.env` files. Use frontend environment variables *only* for public endpoints (e.g., public API URLs).

---

## 5. Securing Node.js Backends

A Node.js backend does not automatically expose `.env` files unless misconfigured. Follow these rules to avoid common security pitfalls:

### 1. Avoid Serving Root Static Directories
Never point static middleware like `express.static()` to your application root directory:

```javascript
// ❌ DANGEROUS: Exposes .env and all project source files to the web
app.use(express.static(__dirname));

// ✅ SAFE: Only serve public assets from a subfolder
app.use(express.static(path.join(__dirname, 'public')));
```

### 2. Process Management & Environment Injection
Load `.env` files using `dotenv` inside your backend entry point:

```javascript
require('dotenv').config();

const dbPassword = process.env.DB_PASSWORD; // Stored in system memory, not web-accessible
```

Use a process manager like **PM2** to manage your Node.js application process in production:

```bash
# Install PM2
npm install -g pm2

# Start process
pm2 start server.js --name "my-backend"

# Save configuration to persist across restarts
pm2 save
pm2 startup
```

---

## 6. Real-Time Log Monitoring & Audit Commands

To inspect real-time traffic and detect scanning activity on your server:

```bash
# View live Nginx access log
sudo tail -f /var/log/nginx/access.log

# Search logs for 404 scanning attempts
sudo grep " 404 " /var/log/nginx/access.log

# Monitor system authentication attempts
sudo tail -f /var/log/auth.log
```