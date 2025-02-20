# Nginx Setup and Configuration

This guide covers the installation and configuration of Nginx as a reverse proxy for your Node.js applications.

## 1. Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

## 2. Configure Nginx

### Configuration Guide

1. **Main Nginx Configuration (`nginx.conf`)**:
   - `worker_processes`: Number of worker processes (auto = one per CPU core)
   - `worker_connections`: Maximum simultaneous connections per worker
   - `multi_accept`: Allow each worker to accept multiple connections
   - `keepalive_timeout`: How long to keep idle connections open
   - `server_tokens`: Disable version display in error pages
   - `client_max_body_size`: Maximum allowed size of client requests (important for file uploads)

2. **SSL Configuration Parameters**:
   - `ssl_protocols`: Enabled SSL/TLS protocol versions (TLSv1.2 and TLSv1.3 recommended)
   - `ssl_prefer_server_ciphers`: Prefer server's cipher suite order
   - `ssl_ciphers`: List of allowed cipher suites (ordered by preference)
   - `ssl_certificate`: Path to SSL certificate file
   - `ssl_certificate_key`: Path to SSL private key file

3. **Security Headers**:
   - `X-Frame-Options`: Prevents clickjacking attacks
   - `X-XSS-Protection`: Basic XSS prevention in older browsers
   - `X-Content-Type-Options`: Prevents MIME-type sniffing
   - `Content-Security-Policy`: Controls allowed content sources
   - `Strict-Transport-Security`: Forces HTTPS connections
   - `Referrer-Policy`: Controls referrer information in headers

4. **Proxy Configuration (for Node.js apps)**:
   - `proxy_pass`: URL of upstream Node.js application
   - `proxy_http_version`: HTTP protocol version for proxy
   - `proxy_set_header`: Custom headers to pass to the application:
     - `Upgrade`, `Connection`: Required for WebSocket support
     - `Host`: Original host header
     - `X-Real-IP`: Client's real IP
     - `X-Forwarded-For`: Client IP chain
     - `X-Forwarded-Proto`: Original protocol (HTTP/HTTPS)

5. **Performance Settings**:
   - `sendfile`: Enables kernel sendfile for better file serving
   - `tcp_nopush`, `tcp_nodelay`: TCP optimization options
   - `gzip`: Compression settings:
     - `gzip_comp_level`: Compression level (1-9, 6 recommended)
     - `gzip_types`: MIME types to compress
     - `gzip_vary`: Sends Vary header for proper caching

6. **Timeout Settings**:
   - `proxy_connect_timeout`: Time to establish connection
   - `proxy_send_timeout`: Time between two write operations
   - `proxy_read_timeout`: Time between two read operations
   - `client_body_timeout`: Time to receive client body
   - `client_header_timeout`: Time to receive client headers

7. **Static File Handling**:
   - `expires`: Browser cache duration for static files
   - `add_header Cache-Control`: Additional caching directives
   - `alias`: Path to static files directory

For detailed Nginx configuration options, refer to: https://nginx.org/en/docs/

Create directory for site configurations:
```bash
sudo mkdir -p /etc/nginx/sites-available
sudo mkdir -p /etc/nginx/sites-enabled
```

Create main Nginx configuration:
```bash
sudo nano /etc/nginx/nginx.conf
```

Add this configuration:
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    ##
    # Basic Settings
    ##
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    # MIME
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # SSL Settings
    ##
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    ##
    # Logging Settings
    ##
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    ##
    # Gzip Settings
    ##
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;

    ##
    # Virtual Host Configs
    ##
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

## 3. Configure Default Site

Create default site configuration:
```bash
sudo nano /etc/nginx/sites-available/default
```

Add this configuration:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Root directory and index files
    root /var/www/html;
    index index.html index.htm;

    # Default location block
    location / {
        try_files $uri $uri/ =404;
    }
}
```

## 4. Configure Node.js Application Proxy

Create configuration for your Node.js application:
```bash
sudo nano /etc/nginx/sites-available/nodejs-app
```

Add this configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/your-domain.com.crt;
    ssl_certificate_key /etc/nginx/ssl/your-domain.com.key;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Proxy to Node.js application
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Static files location
    location /static/ {
        alias /var/www/your-app/static/;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/nodejs-app /etc/nginx/sites-enabled/
```

## 5. SSL Certificate Setup

### Using AWS Certificate Manager (ACM)

1. **Request Certificate in ACM**:
   - Go to AWS Certificate Manager in your AWS Console
   - Click "Request a certificate"
   - Choose "Request a public certificate"
   - Enter your domain name (e.g., `example.com`)
   - Add additional domains if needed (e.g., `*.example.com` for wildcard)
   - Choose "DNS validation" (recommended) or "Email validation"
   - Add relevant tags if needed
   - Click "Request"

2. **DNS Validation**:
   - If using Route 53, click "Create records in Route 53"
   - Wait a few minutes for the certificate to be validated
   - Status will change to "Issued" when ready

3. **Download and Install ACM Certificate**:
   ```bash
   # Create directory for certificates
   sudo mkdir -p /etc/nginx/ssl
   cd /etc/nginx/ssl
   
   # Install AWS CLI if not already installed
   sudo apt install -y awscli
   
   # Configure AWS CLI with your credentials
   aws configure
   # Enter your AWS Access Key ID
   # Enter your AWS Secret Access Key
   # Enter your default region
   
   # Download the certificate from ACM
   # Replace the values in <> with your actual values
   aws acm export-certificate \
       --certificate-arn <your-certificate-arn> \
       --region <your-region> \
       --output text \
       > /etc/nginx/ssl/your-domain.com.crt
   
   # Save the private key
   sudo nano /etc/nginx/ssl/your-domain.com.key
   # Paste your private key here and save (Ctrl+X, then Y)
   
   # Set proper permissions
   sudo chmod 600 /etc/nginx/ssl/your-domain.com.*
   ```

4. **Update Node.js Application Nginx Configuration**:
   ```bash
   # Open the Node.js application Nginx configuration
   sudo nano /etc/nginx/sites-available/nodejs-app
   ```

   Replace or update the SSL configuration section:
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;
       # Redirect all HTTP requests to HTTPS
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name your-domain.com;  # Replace with your actual domain

       # Updated ACM Certificate configuration
       ssl_certificate     /etc/nginx/ssl/your-domain.com.crt;
       ssl_certificate_key /etc/nginx/ssl/your-domain.com.key;
       
       # Strong SSL Security Settings
       ssl_session_timeout 1d;
       ssl_session_cache shared:SSL:50m;
       ssl_session_tickets off;
       
       # Modern configuration
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
       ssl_prefer_server_ciphers off;
       
       # HSTS (uncomment if you're sure)
       # add_header Strict-Transport-Security "max-age=63072000" always;

       # Rest of your existing configuration (proxy_pass, etc.)
       location / {
           proxy_pass http://localhost:3000;
           # ... rest of your proxy configuration ...
       }
   }
   ```

5. **Test and Reload Nginx**:
   ```bash
   # Test the configuration
   sudo nginx -t

   # If test is successful, reload Nginx
   sudo systemctl reload nginx
   ```

6. **Verify SSL Setup**:
   ```bash
   # Test HTTPS
   curl -I https://your-domain.com

   # You can also verify using online SSL checker tools like:
   # - SSL Labs (https://www.ssllabs.com/ssltest/)
   # - DigiCert SSL Checker (https://www.digicert.com/help/)
   ```

### Alternative: Using Let's Encrypt (for non-AWS setups)
```