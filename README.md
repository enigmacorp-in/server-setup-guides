# Server Setup Guide

This guide provides step-by-step instructions for setting up a production server environment on EC2. The setup includes:

- Initial server security and access setup
- Node.js environment using n package manager
- Docker containers for MySQL and MongoDB
- Nginx as reverse proxy
- PM2 for process management

## Initial Server Setup Steps

1. **SSH Key Setup**
   ```bash
   # On your local machine, if you need to generate a new key pair
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   
   # Copy your public key to the server
   ssh-copy-id -i ~/.ssh/id_rsa.pub user@your-server-ip
   
   # Alternative manual method if ssh-copy-id is not available
   cat ~/.ssh/id_rsa.pub | ssh user@your-server-ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
   ```

2. **Basic Security Setup**
   - Update system packages
   - Configure firewall
   - Set up fail2ban
   - Secure SSH configuration

## Components We'll Install

1. **Node.js Environment**
   - n package manager for Node.js version management
   - Node.js v14.x

2. **Database Containers**
   - MySQL in Docker
   - MongoDB in Docker

3. **Web Server**
   - Nginx for reverse proxy
   - SSL/TLS configuration

4. **Process Management**
   - PM2 for Node.js applications

## Directory Structure
```
/home/ubuntu/
├── apps/                    # Application directories
├── docker/                  # Docker compose files and configurations
│   ├── mysql/
│   └── mongodb/
└── nginx/                   # Nginx configuration files
```

## Next Steps

Detailed setup instructions for each component are provided in separate files:
- [Initial Security Setup](docs/01-security-setup.md)
- [Node.js Installation](docs/02-nodejs-setup.md)
- [Docker & Database Setup](docs/03-docker-database-setup.md)
- [Nginx Configuration](docs/04-nginx-setup.md)
- [PM2 Setup](docs/05-pm2-setup.md) 